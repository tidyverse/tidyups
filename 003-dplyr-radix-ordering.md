
# Tidyup 3: Radix Ordering in `dplyr::arrange()`

**Champion**: Davis

**Status**: Proposal

## Abstract

As of dplyr 1.0.6, `arrange()` uses a modified version of
`base::order()` to sort columns of a data frame. Recently, vctrs has
gained `vec_order()`, a fast radix ordering function with first class
support for data frames and custom vctrs types, along with enhanced
character ordering. The purpose of this tidyup is to propose switching
`arrange()` from `order()` to `vec_order()` in the most user-friendly
way possible.

## Motivation

Thanks to the data.table team, R &gt;= 3.3 gained support for an
extremely fast radix ordering algorithm in `order()`. This has become
the default algorithm for ordering most atomic types, with the notable
exception of character vectors. Radix ordering character vectors is only
possible in the C locale, but the shell sort currently in use by
`order()` respects the system locale. Because R is justifiably hesitant
to break backwards compatibility, if *any* character vector is present,
the entire ordering procedure falls back to a much slower shell sort.
Because dplyr uses `order()` internally, the performance of `arrange()`
is negatively affected by this fallback.

Inspired by the performance of the radix ordering algorithm, and by its
many practical applications for data science, a radix order-based
`vec_order()` was added to vctrs, which has the following benefits:

-   Radix ordering on all atomic types.

-   First class support for data frames, including specifying sorting
    direction on a per column basis.

-   Support for an optional character transformation function, which
    generates an intermediate *sort key*. When sorted in the C locale,
    the sort key returns an ordering that would be equivalent to sorting
    the original vector in an alternate locale.

``` r
library(stringi)
vec_order <- vctrs:::vec_order_radix
set.seed(123)

# 10,000 random strings, sampled to a total size of 1,000,000
n_unique <- 10000L
n_total <- 1000000L

dictionary <- stri_rand_strings(n = n_unique, length = sample(1:30, n_unique, replace = TRUE))
x <- sample(dictionary, size = n_total, replace = TRUE)

head(x)
```

    ## [1] "vW5VN"                          "qdNNzemEw1sXdoaqsLz1mJc3bGuixU"
    ## [3] "mljKvuznJRP"                    "22wLX7L"                       
    ## [5] "wcIz5PS93kRUC"                  "2yy09KfokjQoBwumnUascCD"

``` r
# Force `order()` to use the C locale to match `vec_order()`
bench::mark(
  base = withr::with_locale(c(LC_COLLATE = "C"), order(x)),
  vctrs = vec_order(x)
)
```

    ## # A tibble: 2 x 6
    ##   expression      min   median `itr/sec` mem_alloc `gc/sec`
    ##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    ## 1 base          1.11s    1.11s     0.904    3.89MB      0  
    ## 2 vctrs       14.26ms  16.25ms    56.7      12.7MB     23.9

``` r
# Force `vec_order()` to use the American English locale, which is also
# my system locale. To do that we'll need to generate a sort key, which
# we can sort in the C locale, but the result will be like we sorted
# directly in the American English locale.
stri_sort_key(head(x), locale = "en_US")
```

    ## [1] "TV\\x1dTD\\x01\\x09\\x01\\xc5\\xe0\\xc5\\xdc\\xdc"                                                                    
    ## [2] "J0DD\\2B2V\\x15NX0F*JN@\\\\x15B<.\\x19,6R:XR\\x01\"\\x01\\xc4\\xdc\\xdc\\xc3\\xdc\\xc3\\xdc\\xc1\\xdc\\xc3\\xdc\\xc3\\xdc\\xc3\\xdc\0"
    ## [3] "B@<>TR\\D<LH\\x01\\x0f\\x01\\xc3\\xdc\\xc2\\xdc\\xdc\\xdc"                                                             
    ## [4] "\\x17\\x17V@X!@\\x01\\x0b\\x01\\xc3\\xdc\\xdc\\xc5\\xdc"                                                               
    ## [5] "V.:\\\\x1dHN%\\x19>LR.\\x01\\x11\\x01\\xc4\\xdc\\xc4\\xdc\\xdc\\xc3\\xdc\\xdc\\xdc"                                         
    ## [6] "\\x17ZZ\\x13%>4F><JF,VRBDR*N..0\\x01\\x1b\\x01\\xc1\\xdc\\xc2\\xe0\\xc5\\xdc\\xc2\\xdc\\xc3\\xdc\\xdc"

``` r
Sys.getlocale("LC_COLLATE")
```

    ## [1] "en_US.UTF-8"

``` r
bench::mark(
  base = order(x),
  vctrs = vec_order(x, chr_transform = ~stri_sort_key(.x, locale = "en_US"))
)
```

    ## Warning: Some expressions had a GC in every iteration; so filtering is disabled.

    ## # A tibble: 2 x 6
    ##   expression      min   median `itr/sec` mem_alloc `gc/sec`
    ##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    ## 1 base           5.5s     5.5s     0.182    3.81MB     0   
    ## 2 vctrs       586.4ms  586.4ms     1.71    20.18MB     1.71

``` r
bench::mark(
  sort_key = stri_sort_key(x, locale = "en_US")
)
```

    ## # A tibble: 1 x 6
    ##   expression      min   median `itr/sec` mem_alloc `gc/sec`
    ##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    ## 1 sort_key      594ms    594ms      1.68    7.63MB        0

In dplyr, we’d like to utilize `vec_order()` while breaking as little
code as possible. Switching to `vec_order()` has many potential positive
impacts, including:

-   Improved performance when ordering character vectors.

    -   Improved overall performance when ordering character vectors
        alongside other atomic types.

-   Improved reproducibility across sessions by removing a dependency on
    an environment variable, `LC_COLLATE`, which is used by `order()` to
    determine the locale.

-   Improved reproducibility across OSes where `LC_COLLATE` might
    differ, even for the same locale. For example, `"en_US"` on a Mac is
    approximately equivalent to `"English_United States.1252"` on
    Windows.

-   Improved consistency within the tidyverse. Specifically, with
    stringr, which defaults to the `"en"` locale and has an explicit
    argument for changing the locale,
    i.e. `stringr::str_order(locale = "en")`.

However, making this switch also has potential negative impacts:

-   Breaking code that relies strongly on the ordering resulting from
    using the `LC_COLLATE` environment variable.

-   Surprising users if the new ordering does not match the previous
    one.

## Proposal

To switch to `vec_order()` internally while surprising the least amount
of users, it is proposed that the data frame method for `arrange()` gain
a new argument, `.locale`, with the following properties:

-   Default to `NULL`

    -   If stringi is installed, the American English locale, `"en"`,
        will be assumed.

    -   Otherwise, the C locale will be used, with a warning informing
        the user and encouraging them to silence the warning by either
        installing stringi or specifying to use the C locale explicitly.

-   If stringi is installed, allow a single locale identifier, such as
    `"fr"`, for explicitly adjusting the locale to order with. If
    stringi is not installed, and the user has explicitly specified a
    locale identifier, an error will be thrown.

-   Allow `"C"` to be specified as a special case, which is an explicit
    way to request the C locale. If the exact details of the ordering
    are not critical, this is often much faster than specifying a locale
    identifier.

This proposal relies on `stringi::stri_sort_key()`, which generates the
sort key mentioned under Motivation as a proxy that can be ordered in
the C locale. However, sort key generation is expensive. In fact, it is
often the most expensive part of the entire process. That said,
generating a sort key + sorting it in the C locale is generally still
5-10x faster than using `order()` directly. If performance is critical,
users can specify `.locale = "C"` to get the maximum benefits of radix
ordering.

## Reference Implementation

-   Using `vec_order()` in `arrange()`, and adding `.locale`

    -   <https://github.com/tidyverse/dplyr/pull/5868>

-   Renaming `vec_order_radix()` to `vec_order()`

    -   <https://github.com/r-lib/vctrs/pull/1375>

## Backwards Compatibility

### arrange()

The proposal outlined above is purposefully as conservative as possible,
preserving the results of programs using the American English locale,
which is the most widely used locale in R, while sacrificing a bit of
performance from the generation of the sort key.

This proposal will impact non-English Latin script languages. For
example, in a majority of Latin script languages, including `"en"`, ø
(like *eu* in the French word *bleu*) sorts after o, but before
p. However, a few languages, such as Danish, sort ø as a unique
character after z. Danish users that have `LC_COLLATE` set to Danish may
be surprised that `arrange()` would be placing ø in the “wrong order”
even though they have set `LC_COLLATE` . The fix would be to set
`.locale = "da"` in their calls to `arrange()`.

``` r
sort_key_en <- function(x) stri_sort_key(x, locale = "en")
sort_key_da <- function(x) stri_sort_key(x, locale = "da")

x <- c("ø", "o", "p", "z")

x[vec_order(x, chr_transform = sort_key_en)]
```

    ## [1] "o" "ø" "p" "z"

``` r
x[vec_order(x, chr_transform = sort_key_da)]
```

    ## [1] "o" "p" "z" "ø"

TODO: How exactly does this break non-Latin languages? Chinese?
Japanese?

### arrange\_at/if/all()

These three variants of `arrange()` are superseded, and would not gain a
`.locale` argument. They would inherit the `arrange()` default of
`.locale = NULL` . Note that they would always warn about `.locale`
falling back to C if stringi is not installed, with no way for the user
to silence this by specifying `.locale = "C"` - *so it may be worth
adding `.locale` anyways*, or just setting the default to `"C"`.

### Other order-related functions

Within dplyr, there are two other direct places that `vec_order()` is
called, `with_order()` and `grouped_df()`.

`with_order()`, and through it, `order_by()`, `lag(order_by =)`, and
`lead(order_by =)`, would all be affected by changing to a `vec_order()`
that defaulted to the C locale. However, it is relatively uncommon to
use the two window function helpers of `with_order()` and `order_by()`
to order by a character vector. Additionally, it is also being proposed
that rewrites of `lag()` and `lead()` drop their `order_by` arguments
altogether. Because of this, no changes would be made to these functions
to preserve backwards compatibility, and they would begin ordering
character vectors in the C locale.

`grouped_df()`, and through it, `group_by()` and many other grouping
variants and methods, would also be affected from a switch to ordering
in the C locale. The details related to this switch are complex enough
for their own tidyup, and will not be discussed here.

## Rationale and Alternatives

### Defaulting to the C locale

One alternative to the above proposal is to default `arrange()` to the C
locale, while still allowing users to specify `.locale` for ordering in
alternative locales.

-   This has the benefit of making it clearer that stringi is an
    optional dependency, which would only be used if a user requests a
    specific locale identifier. The default behavior would never require
    stringi.

-   Additionally, the performance improvements would be even more
    substantial since no sort key would be required by default.

-   Would also be consistent with `group_by()` if that started to use
    the C locale by default.

-   However, this would have the potential to alter nearly every call to
    `arrange()`, since the C locale is not identical to the American
    English locale. In particular, in the C locale all capital letters
    are ordered before lowercase ones, such as: `c(A, B, a, b)` , while
    in the American English locale letters are first grouped together
    regardless of capitalization, and then lowercase letters are placed
    before uppercase ones, like: `c(a, A, b, B)`. This may look like a
    small difference, but it is enough to justify not defaulting to the
    C locale.

-   Also, not as consistent with stringr, which defaults to `"en"`.

### Tagged character vectors

A second proposed alternative was to implement a “tagged” character
vector class, with an additional attribute specifying the locale to be
used when ordering. This would remove the need for any `.locale`
argument, and the locale would even be allowed to vary between columns.
If no locale tag was supplied, `arrange()` would default to either
`"en"` or `"C"` for the locale. This approach is relatively clean, but
is practically very difficult because it would require cross package
effort to generate and retain these locale tags. Additionally, it
doesn’t solve the problem of avoiding breakage for existing code that
uses a non-English locale. Lastly, it would require an additional
learning curve for users to understand how to use them in conjunction
with `arrange()`.

## Changelog

### 2021-06-14

The tidyverse group meeting of 2021-06-14 resulted in a number of new
discussion points. In particular, it was discussed that forcing the
American English locale with no way to globally revert back to the old
behavior was probably a bit too aggressive. To solve this, we are
considering making a tidyverse wide environment/global variable that can
influence the locale used by `arrange()`, while still keeping an English
default. This would be an exception to our general process of avoiding
global options that can affect computation, but might be worth it in
this case since an English default could be extremely annoying to
non-English users.

It is worth considering whether this option should affect only dplyr, or
if it should also affect stringr’s default. Additionally, it could
affect readr and possibly lubridate/clock (if this locale also affects
month names / weekday names).

We could introduce one of the following options:

-   `TIDYVERSE_LOCALE` to control all things locale across the
    tidyverse.

-   `DPLYR_LOCALE` to limit locale behavior to dplyr.

-   `TIDYVERSE_COLLATE` to limit it to only sorting behavior, as opposed
    to date-specific options controlled by `LC_TIME`, like the month or
    weekday names.

Another option is to provide `.locale = "legacy"` which would fall back
to using `base::order()` as `arrange()` currently does.

The group also mentioned a few other non-English languages where an
English default might be impactful: French, German, Czech, Danish.
