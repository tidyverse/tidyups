
# Tidyup 8: Retaining and excluding rows

**Champion**: Davis

**Co-Champion**: Hadley

**Status**: Draft

``` r
library(dplyr, warn.conflicts = FALSE)
```

## Abstract

`dplyr::filter()` has been a core dplyr verb since the very beginning,
but over the years we’ve isolated a few key issues with it:

- The name `filter()` is ambiguous, are you retaining rows or excluding
  them? i.e., are you filtering *for* rows or filtering *out* rows?

- `filter()` is optimized for the case of *retaining* rows, but you are
  just as likely to try and use it for *excluding* rows. Using
  `filter()` to exclude rows forces you to confront complex boolean
  logic and explicitly handle missing values, which is difficult to
  teach, error prone to write, and hard to understand when you come back
  to it in the future.

- `filter()` combines `,` separated conditions with `&` because this
  covers the majority of the cases. But if you’d like to combine
  conditions with `|`, then you have to introduce parentheses around
  your conditions and combine them into one large condition separated by
  `|`, reducing readability.

## Solution

To address these issues, we propose two new families of dplyr verbs:

``` r
# Data frame functions
retain(.data, ..., .by = NULL, .missing = FALSE)
exclude(.data, ..., .by = NULL, .missing = FALSE)

# Vector functions
when_any(..., missing = NULL)
when_all(..., missing = NULL)
```

For `retain()` and `exclude()`:

- Both combine conditions with `&`.

- Both treat `NA` as `FALSE`.

  - As we will see, having `exclude()` work in this way simplifies many
    cases of using `filter()` to exclude rows.

  - `.missing = TRUE` opts in to treating `NA` like `TRUE`. For
    `retain()`, this retains missing values. For `exclude()`, this
    excludes missing values.

For `when_any()` and `when_all()`:

- These are equivalents to `pmin()` and `pmax()`, but applied to `any()`
  and `all()`. They could be called `pany()` and `pall()`, but these
  names are friendlier.

- `when_any()` combines conditions with `|`. As we will see, this is
  particularly useful in combination with `retain()`.

- `when_all()` combines conditions with `&`.

- `missing = NULL` propagates `NA` through according to the typical `&`
  and `|` rules. Propagating missing values by default combines well
  `retain()` and `exclude()`.

- These functions can be used anywhere, not just in `retain()` and
  `exclude()`.

- They do have the potential to be confused with `if_any()` and
  `if_all()`, which apply a function to a selection of columns but
  otherwise operate similarly.

Notably, `filter()` would roughly alias to `retain()` and would not be
superseded or deprecated in any way. We recognize that there is too much
existing code out there for this to be possible. That said, all dplyr
documentation would be updated to use `retain()` or `exclude()`.

## Implementation

Try for yourself at:

``` r
pak::pak("tidyverse/dplyr@feature/retain-exclude")
```

Note that this implementation is not finished yet, and may have bugs.

## Motivation

### `filter()` ambiguity

The word “filter” has two meanings depending on the context:

- Filter *for* some values you want to keep

- Filter *out* some values you want to drop

This is unfortunate and can make `filter()` hard to teach. We believe
that the names `retain()` and `exclude()` are much clearer in their
intent:

``` r
data <- tibble(
  account = c(100, 50, 20, 12, 70)
)

# Is this keeping rows where `account > 20` or dropping them?
data |> filter(account > 20)
```

    ## # A tibble: 3 × 1
    ##   account
    ##     <dbl>
    ## 1     100
    ## 2      50
    ## 3      70

``` r
# These are very clear
data |> retain(account > 20)
```

    ## # A tibble: 3 × 1
    ##   account
    ##     <dbl>
    ## 1     100
    ## 2      50
    ## 3      70

``` r
data |> exclude(account > 20)
```

    ## # A tibble: 2 × 1
    ##   account
    ##     <dbl>
    ## 1      20
    ## 2      12

### Excluding rows using `this & that`

Due to the lack of an explicit “exclude rows” function in dplyr, many
people turn to `filter()`. But using `filter()` in this way often
requires replacing `,` separated conditions with `&` separated
conditions, wrapping the whole thing in parentheses, and then prefixing
it all with a `!`.

Take a look at this example. Our goal is:

> *Exclude* rows where the patient is deceased *and* the year of death
> was before 2012.

``` r
patients <- tibble(
  name = c("Anne", "Mark", "Sarah", "Davis", "Max", "Derek"),
  deceased = c(FALSE, TRUE, FALSE, TRUE, TRUE, FALSE),
  date = c(2005, 2010, 2013, 2020, 2010, 2000)
)

patients
```

    ## # A tibble: 6 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Mark  TRUE      2010
    ## 3 Sarah FALSE     2013
    ## 4 Davis TRUE      2020
    ## 5 Max   TRUE      2010
    ## 6 Derek FALSE     2000

With `filter()`:

``` r
patients |>
  filter(!(deceased & date < 2012))
```

    ## # A tibble: 4 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Sarah FALSE     2013
    ## 3 Davis TRUE      2020
    ## 4 Derek FALSE     2000

Compare that with this proposed usage of `exclude()`:

``` r
patients |>
  exclude(deceased, date < 2012)
```

    ## # A tibble: 4 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Sarah FALSE     2013
    ## 3 Davis TRUE      2020
    ## 4 Derek FALSE     2000

Note how we drop:

- The `!`
- The `()`
- The `&` (in favor of sticking with `,`)

This results in an `exclude()` statement that precisely matches the
intent of the original problem statement. In other words, you can “write
it like you say it”.

In general, the following form of `filter()` can always be written as a
much simpler `exclude()`:

``` r
data |> filter(!(this & that & those))
data |> exclude(this, that, those)
```

#### Missing value handling

Let’s introduce some missing values to `patients`.

``` r
patients <- tibble(
  name = c("Anne", "Mark", "Sarah", "Davis", "Max", "Derek", "Tina"),
  deceased = c(FALSE, TRUE, NA, TRUE, NA, FALSE, TRUE),
  date = c(2005, 2010, NA, 2020, 2010, NA, NA)
)

patients
```

    ## # A tibble: 7 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Mark  TRUE      2010
    ## 3 Sarah NA          NA
    ## 4 Davis TRUE      2020
    ## 5 Max   NA        2010
    ## 6 Derek FALSE       NA
    ## 7 Tina  TRUE        NA

Our goal before was:

> Exclude rows where the patient is deceased and the year of death was
> before 2012.

In the data above, that looks to just be row 2. Let’s try the same
`filter()` from before:

``` r
patients |>
  filter(!(deceased & date < 2012))
```

    ## # A tibble: 3 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Davis TRUE      2020
    ## 3 Derek FALSE       NA

This drops 4 rows from our dataset! Let’s see which ones:

``` r
patients |>
  mutate(to_retain = !(deceased & date < 2012)) |>
  mutate(what_filter_sees = to_retain & !is.na(to_retain))
```

    ## # A tibble: 7 × 5
    ##   name  deceased  date to_retain what_filter_sees
    ##   <chr> <lgl>    <dbl> <lgl>     <lgl>           
    ## 1 Anne  FALSE     2005 TRUE      TRUE            
    ## 2 Mark  TRUE      2010 FALSE     FALSE           
    ## 3 Sarah NA          NA NA        FALSE           
    ## 4 Davis TRUE      2020 TRUE      TRUE            
    ## 5 Max   NA        2010 NA        FALSE           
    ## 6 Derek FALSE       NA TRUE      TRUE            
    ## 7 Tina  TRUE        NA NA        FALSE

Because `filter()` treats `NA` as `FALSE`, we unexpectedly drop *more
than we expected*.

This phenomenon is rather confusing, but is due to the fact that
`filter()` is designed around the idea that you’re going to tell it
which rows to *keep*. In that case, ignoring `NA`s makes sense, i.e. if
you don’t *know* that you want to keep that row (because an `NA` is
ambiguous), then you probably don’t want to keep it.

This works well until you try to use `filter()` as a way to *exclude*
rows, at which point this behavior works against you. At this point,
most people reach for `is.na()` to help them out. Here’s what a
`filter()` call that only excludes rows where you *know* the patient is
deceased and the year of death was before 2012 might look like:

``` r
patients |>
  filter(
    !((deceased & !is.na(deceased)) &
      (date < 2012 & !is.na(date)))
  )
```

    ## # A tibble: 6 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Sarah NA          NA
    ## 3 Davis TRUE      2020
    ## 4 Max   NA        2010
    ## 5 Derek FALSE       NA
    ## 6 Tina  TRUE        NA

That’s horrible! Advanced users of dplyr might think about this for a
moment and rewrite as:

``` r
patients |>
  filter(!coalesce(deceased & date < 2012, FALSE))
```

    ## # A tibble: 6 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Sarah NA          NA
    ## 3 Davis TRUE      2020
    ## 4 Max   NA        2010
    ## 5 Derek FALSE       NA
    ## 6 Tina  TRUE        NA

But that’s still pretty confusing, took a lot of time to get there, and
you’ll likely look back on this in a year wondering what you were doing
with that `coalesce()`. Here’s what using `exclude()` would look like:

``` r
patients |>
  exclude(deceased, date < 2012)
```

    ## # A tibble: 6 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Sarah NA          NA
    ## 3 Davis TRUE      2020
    ## 4 Max   NA        2010
    ## 5 Derek FALSE       NA
    ## 6 Tina  TRUE        NA

Just like with `filter()` or `retain()`, `exclude()` treats `NA` values
as `FALSE`. The difference is that `exclude()` expects that you are
going to tell it which rows to *exclude* (rather than which rows to
retain), so the default behavior of treating `NA` like `FALSE` works
*with you* rather than *against you*. It’s also much easier to
understand when you look back on it a year from now!

### Excluding rows using `this | that`

Let’s look at another example. Our goal is:

> *Exclude* rows where class is “suv” *or* mpg is less than 15.

``` r
cars <- tibble(
  class = c("suv", "suv", "suv", "coupe", "coupe", "coupe", NA, NA, NA),
  mpg = c(10, 20, NA, 10, 20, NA, 10, 20, NA)
)

cars
```

    ## # A tibble: 9 × 2
    ##   class   mpg
    ##   <chr> <dbl>
    ## 1 suv      10
    ## 2 suv      20
    ## 3 suv      NA
    ## 4 coupe    10
    ## 5 coupe    20
    ## 6 coupe    NA
    ## 7 <NA>     10
    ## 8 <NA>     20
    ## 9 <NA>     NA

Because dplyr doesn’t have a way to exclude rows, you’d reach for
`filter()`. You’d probably first try to translate the problem statement
directly to code and then invert it with a `!` in the front:

``` r
cars |>
  filter(!(class == "suv" | mpg < 15))
```

    ## # A tibble: 1 × 2
    ##   class   mpg
    ##   <chr> <dbl>
    ## 1 coupe    20

Hmm, we’ve got two issues here:

- We’re again facing a scenario where we need to add boolean operators
  into the mix.

- We’re again mistakenly dropping rows with `NA`s where we don’t *know*
  the conditions were true.

A correct `filter()` solution would look like:

``` r
cars |>
  filter(
    !((class == "suv" & !is.na(class)) |
      (mpg < 15 & !is.na(mpg)))
  )
```

    ## # A tibble: 4 × 2
    ##   class   mpg
    ##   <chr> <dbl>
    ## 1 coupe    20
    ## 2 coupe    NA
    ## 3 <NA>     20
    ## 4 <NA>     NA

An equally correct solution would be to recognize that you can split
this into two filter statements using [De Morgan’s
Laws](https://en.wikipedia.org/wiki/De_Morgan%27s_laws).

``` r
cars |>
  filter(class != "suv" | is.na(class)) |>
  filter(mpg >= 15 | is.na(mpg))
```

    ## # A tibble: 4 × 2
    ##   class   mpg
    ##   <chr> <dbl>
    ## 1 coupe    20
    ## 2 coupe    NA
    ## 3 <NA>     20
    ## 4 <NA>     NA

This is simpler, but has *a lot* of mental overhead. Remember that the
original problem statement was:

> *Exclude* rows where class is “suv” *or* mpg is less than 15.

To achieve this we’ve had to:

- Flip the conditions (taking special care that `< 15` is now `>= 15`)
- Specially handle missing values
- Introduce `|`, at a minimum, or `!`, `()`, and `&` if not flipping the
  conditions

Here’s the `exclude()` solution:

``` r
cars |>
  exclude(class == "suv" | mpg < 15)
```

    ## # A tibble: 4 × 2
    ##   class   mpg
    ##   <chr> <dbl>
    ## 1 coupe    20
    ## 2 coupe    NA
    ## 3 <NA>     20
    ## 4 <NA>     NA

Note how this *precisely* translates the problem statement into code.
Also note how the `NA` behavior is correct out-of-the-box with no
additional adjustments.

One beautiful thing about `exclude()` is that any `|` separated
conditions can always be written as two sequential `exclude()`
statements, meaning that you could also remove the `|` to simplify this
further as:

``` r
cars |>
  exclude(class == "suv") |>
  exclude(mpg < 15)
```

    ## # A tibble: 4 × 2
    ##   class   mpg
    ##   <chr> <dbl>
    ## 1 coupe    20
    ## 2 coupe    NA
    ## 3 <NA>     20
    ## 4 <NA>     NA

In general, the following form of `filter()` can always be written as a
much simpler `exclude()`:

``` r
data |> filter(!(this | that | those))
data |> exclude(this) |> exclude(that) |> exclude(those)
```

#### With `anti_join()`

Savvy dplyr users might recognize that the above `filter()` could also
be expressed as an anti join to avoid dealing with missing values:

``` r
anti_join(
  cars,
  cars |> filter(class == "suv" | mpg < 15),
  by = join_by(class, mpg)
)
```

    ## # A tibble: 4 × 2
    ##   class   mpg
    ##   <chr> <dbl>
    ## 1 coupe    20
    ## 2 coupe    NA
    ## 3 <NA>     20
    ## 4 <NA>     NA

I’d argue that this is much “too clever” for the problem at hand. It
also gets prohibitively slow as the size of the data increases because
you are anti-joining across all columns at once.

#### `tidyr::drop_na()`

`tidyr::drop_na()` is a special case of `exclude(this | that)`. Because
dplyr didn’t have an “exclude rows” solution, years ago we added
`tidyr::drop_na()` as a way to drop rows where *any* specified column is
`NA`. The goal here is:

> *Exclude* rows where `deceased` *or* `date` are missing.

``` r
patients <- tibble(
  name = c("Anne", "Mark", "Sarah", "Davis", "Max", "Derek"),
  deceased = c(FALSE, NA, FALSE, TRUE, NA, FALSE),
  date = c(2005, 2010, NA, 2020, 2010, 2000)
)

patients |> tidyr::drop_na(deceased, date)
```

    ## # A tibble: 3 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Davis TRUE      2020
    ## 3 Derek FALSE     2000

``` r
# Equivalent `filter()`s
patients |> filter(!(is.na(deceased) | is.na(date)))
```

    ## # A tibble: 3 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Davis TRUE      2020
    ## 3 Derek FALSE     2000

``` r
patients |> filter(!if_any(c(deceased, date), is.na))
```

    ## # A tibble: 3 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Davis TRUE      2020
    ## 3 Derek FALSE     2000

With `exclude()`, you can express this as sequential `exclude()` calls
if you just have 2-3 columns to work with:

``` r
patients |>
  exclude(is.na(deceased)) |>
  exclude(is.na(date))
```

    ## # A tibble: 3 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Davis TRUE      2020
    ## 3 Derek FALSE     2000

Or, if you have many columns, you can use `if_any()` like with the
`filter()` example above, but in a more readable form:

``` r
patients |> exclude(if_any(c(deceased, date), is.na))
```

    ## # A tibble: 3 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Davis TRUE      2020
    ## 3 Derek FALSE     2000

### Retaining rows using `this | that`

We’ve talked a lot about *excluding* rows, but this tidyup also proposes
a feature that helps with *retaining* rows when using conditions
combined with `|` - `when_any()`.

Our goal here is:

> *Retain* rows for “US” and “CA” when the score is between 200-300,
> *or* rows for “PR” and “RU” when the score is between 100-200.

``` r
countries <- tibble(
  name = c("US", "CA", "PR", "RU", "US", NA, "CA", "PR"),
  score = c(200, 100, 150, NA, 50, 100, 300, 250)
)

countries
```

    ## # A tibble: 8 × 2
    ##   name  score
    ##   <chr> <dbl>
    ## 1 US      200
    ## 2 CA      100
    ## 3 PR      150
    ## 4 RU       NA
    ## 5 US       50
    ## 6 <NA>    100
    ## 7 CA      300
    ## 8 PR      250

Here’s a `filter()` solution, note how we’ve introduced 3 boolean
operators, `&`, `|`, and `()`, decreasing readability and increasing the
mental gymnastics required to understand it:

``` r
countries |>
  filter(
    (name %in% c("US", "CA") & between(score, 200, 300)) |
      (name %in% c("PR", "RU") & between(score, 100, 200))
  )
```

    ## # A tibble: 3 × 2
    ##   name  score
    ##   <chr> <dbl>
    ## 1 US      200
    ## 2 PR      150
    ## 3 CA      300

With `when_any()`, you specify `,` separated conditions like you’re used
to, but they get combined with `|` rather than `&`. This allows us to
reduce the amount of boolean operators introduced down to just `&`, and
it remains pretty readable:

``` r
countries |>
  retain(when_any(
    name %in% c("US", "CA") & between(score, 200, 300),
    name %in% c("PR", "RU") & between(score, 100, 200)
  ))
```

    ## # A tibble: 3 × 2
    ##   name  score
    ##   <chr> <dbl>
    ## 1 US      200
    ## 2 PR      150
    ## 3 CA      300

We think this is a better solution than adding a
`.combine = c("&", "|")` style argument to `retain()` itself, which
feels less readable overall:

``` r
countries |>
  retain(
    name %in% c("US", "CA") & between(score, 200, 300),
    name %in% c("PR", "RU") & between(score, 100, 200),
    .combine = "|"
  ))
```

It’s also nice that `when_any()` and `when_all()` are useful outside of
just `retain()` and `exclude()`, which we would not get with a
`.combine` argument.

#### `when_all()`?

For completeness, `when_all()` also exists as a `pall()`
(“parallel-all”) style operator, but it isn’t as useful with `retain()`
or `exclude()` because they already combine conditions with `&`. It’s
possible there will be cases when it can help you express complex
conditions combining `|` and `&` in a readable way, such as:

``` r
data |>
  retain(
    when_any(this, that),
    when_all(these, those)
  )
```

And it is also worth noting that these can be used outside of `retain()`
and `exclude()`:

``` r
data <- tibble(
  scoreA = c(11, 20, 30, 10, 5),
  scoreB = c(15, 12, 15, 7, 10),
  scoreC = c(30, 28, 30, 31, 24)
)

data |>
  summarise(
    high_scores = sum(when_all(
      scoreA > 10,
      scoreB > 9,
      scoreC > 29
    ))
  )
```

    ## # A tibble: 1 × 1
    ##   high_scores
    ##         <int>
    ## 1           2

``` r
# Compared to this (depending on how defensive you are with parentheses)
data |>
  summarise(
    high_scores = sum(
      (scoreA > 10) &
        (scoreB > 9) &
        (scoreC > 29)
    )
  )
```

    ## # A tibble: 1 × 1
    ##   high_scores
    ##         <int>
    ## 1           2

#### With `exclude()`?

We’ve seen that `when_any()` can be useful with `retain()` because it
allows you to continue specifying your conditions separated by commas
rather than separated by `|` operators and wrapped in `()`. It’s worth
noting that `when_any()` does not add much value to `exclude()` because,
as we saw in the previous section, `exclude()`s involving `|` can be
written as sequential `exclude()`s instead, i.e.:

``` r
data |> exclude(this | that)
data |> exclude(this) |> exclude(that)
```

This isn’t the case for `retain()` and `|`, hence the added value of
`when_any()`.

## TODO: An example for `.missing`?

Is `.missing = TRUE` ever useful in `retain()` and `exclude()`, or was
it just a red herring that is resolved by the fact that we have
`exclude()` now? I seem to remember there are cases in Sarah’s examples
where `.missing = TRUE` would still be useful.

## Backwards compatibility

### `filter()`

`filter()` would alias to `retain()` and would never be superseded or
deprecated. We would be very careful to retain all existing behavior of
`filter()`, but we may decide not to give it new features that
`retain()` and `exclude()` would gain, like `.missing`.

## How to teach

`retain()` and `exclude()` would be taught as a pair. We hope they are
much easier to teach than `filter()` because:

- They have less ambiguous names

- `exclude()` as an explicit way to exclude rows eliminates the need for
  complex boolean logic

- `exclude()` as an explicit way to exclude rows results in intuitive
  missing value behavior

## Additional considerations

### dbplyr

There is a strong connection between `dplyr::filter()` and SQL’s
`FILTER` clause. Moving to `retain()` and `exclude()` would weaken this
connection a bit, but we think that is okay. We already have `mutate()`,
`summarise()`, and `arrange()`, none of which map directly to SQL. Both
`retain()` and `exclude()` would have dbplyr translations directly to
SQL, and dbplyr could likely use much of the existing code already used
for `filter()` to make this happen.

### Alternate names

#### `keep()` and `drop()`

Already taken by `base::drop()` and `purrr::keep()`.

#### `keep_rows()` and `drop_rows()`

The `*_rows()` suffix is nice because it differentiates this operation
from `select()`, which selects *columns*. But if these are to be top
level dplyr verbs, then they need to fit the overall naming scheme.

Proposed top level dplyr verbs:

- `mutate()` / `summarise()`

- `retain()` / `exclude()`

- `arrange()`

- `select()`

With alternate names:

- `mutate()` / `summarise()`

- `keep_rows()` / `drop_rows()`

- `arrange()`

- `select()`

The existing verbs are all single words, and the `_rows()` suffix here
throws off the overall coherence.

## Appendix

### References

Sources of inspiration considered while designing these APIs:

- [kit::pany() and
  kit::pall()](https://cran.r-project.org/web/packages/kit/refman/kit.html#parallel-funs)
- [Stata’s keep if and drop if](https://www.stata.com/manuals/ddrop.pdf)

Related issues and examples:

- <https://github.com/tidyverse/dplyr/issues/6560>

- <https://github.com/tidyverse/dplyr/issues/6891>

### Tables

Tables like these help us ensure there aren’t any holes in our designs.

Intent vs Combine

<table style="width:99%;">
<colgroup>
<col style="width: 15%" />
<col style="width: 15%" />
<col style="width: 67%" />
</colgroup>
<thead>
<tr>
<th>Intent</th>
<th>Combine</th>
<th>Solution</th>
</tr>
</thead>
<tbody>
<tr>
<td>Retain</td>
<td>And</td>
<td><code>retain(a, b, c)</code></td>
</tr>
<tr>
<td>Retain</td>
<td>Or</td>
<td><code>retain(when_any(a, b, c))</code></td>
</tr>
<tr>
<td>Exclude</td>
<td>And</td>
<td><code>exclude(a, b, c)</code></td>
</tr>
<tr>
<td>Exclude</td>
<td>Or</td>
<td><p><code>exclude(when_any(a, b, c))</code></p>
<p>In practice:
<code>exclude(a) |&gt; exclude(b) |&gt; exclude(c)</code></p></td>
</tr>
</tbody>
</table>

Intent vs Missings

| Intent | Missings | Outcome | Usefulness |
|----|----|----|----|
| Retain | Treat as `FALSE` | Retain rows where you *know* the conditions are `TRUE` | Very. Existing `filter()` behavior. |
| Exclude | Treat as `FALSE` | Exclude rows where you *know* the conditions are `TRUE` | Very. Simplifies “treat `filter()` as an exclude” cases. |
| Retain | Treat as `TRUE` | Retain rows where conditions are `TRUE` or `NA` | Unconvinced. Often this is an `exclude()` in disguise. |
| Exclude | Treat as `TRUE` | Exclude rows where conditions are `TRUE` or `NA` | Unconvinced. |
