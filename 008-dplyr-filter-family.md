
# Tidyup 8: Expanding the `filter()` family

**Champion**: Davis

**Co-Champion**: Hadley

**Status**: Draft

``` r
library(dplyr, warn.conflicts = FALSE)
```

## Abstract

`dplyr::filter()` has been a core dplyr verb since the very beginning,
but over the years we’ve isolated a few key issues with it:

- The name `filter()` is ambiguous, are you keeping rows or dropping
  rows? i.e., are you filtering *for* rows or filtering *out* rows?

- `filter()` is optimized for the case of *keeping* rows, but you are
  just as likely to try and use it for *dropping* rows. Using `filter()`
  to drop rows quickly forces you to confront complex boolean logic and
  explicitly handle missing values, which is difficult to teach, error
  prone to write, and hard to understand when you come back to it in the
  future.

- `filter()` combines `,` separated conditions with `&` because this
  covers the majority of the cases. But if you’d like to combine
  conditions with `|`, then you have to introduce parentheses around
  your conditions and combine them into one large condition separated by
  `|`, reducing readability.

## Solution

To address these issues, we propose introducing `filter_out()` as the
complement of `filter()`, and adding two new vector functions to dplyr:

``` r
# Data frame functions
filter(.data, ..., .by = NULL)
filter_out(.data, ..., .by = NULL)

# Vector functions
when_any(..., na_rm = FALSE)
when_all(..., na_rm = FALSE)
```

For `filter()` and `filter_out()`:

- Both combine conditions with `&`.

- Both treat `NA` as `FALSE`.

  - As we will see, having `filter_out()` work in this way simplifies
    many cases of using `filter()` to drop rows.

- A key invariant is that
  `union(filter(data, ...), filter_out(data, ...))` returns `data` (with
  a different row ordering).

  - This is not true of `union(filter(data, ...), filter(data, !(...)))`
    due to the `NA` handling of `filter()`.

  - Put differently, `filter_out()` is the *complement* of `filter()`,
    which is not something that can be said for `filter(!(...))`.

- You can think of `filter_out()` as a *variant* of the core verb,
  `filter()`. This is similar to how `slice_head()` and `slice_tail()`
  are *variants* of the core verb, `slice()`.

For `when_any()` and `when_all()`:

- These are equivalents to `pmin()` and `pmax()`, but applied to `any()`
  and `all()`. They could be called `pany()` and `pall()`, but these
  names are friendlier.

- `when_any()` combines conditions with `|`. As we will see, this is
  particularly useful in combination with `filter()`.

- `when_all()` combines conditions with `&`.

- `na_rm = FALSE` propagates `NA` through according to the typical `&`
  and `|` rules. Propagating missing values by default combines well
  `filter()` and `filter_out()`. `na_rm = TRUE` removes `NA`s “rowwise”
  from the computation, exactly like in `pmin()` and `pmax()`.

- These functions can be used anywhere, not just in `filter()` and
  `filter_out()`.

- They do have the potential to be confused with `if_any()` and
  `if_all()`, which apply a function to a selection of columns but
  otherwise operate similarly.

## Implementation

Try for yourself at:

``` r
pak::pak("tidyverse/dplyr@feature/filter-out")
```

Note that this implementation is not finished yet, and may have bugs.

## Motivation

### `filter()` ambiguity

The word “filter” has two meanings depending on the context:

- Filter *for* some values you want to keep

- Filter *out* some values you want to drop

This is unfortunate and can make `filter()` hard to teach.

While it would be too disruptive to change `filter()`’s name, we hope
that the mere presence of `filter_out()` clears up the ambiguity. By
knowing that `filter_out()` exists and by having it documented alongside
`filter()`, you are more likely to have better intuition about
`filter()` itself.

### Filtering out rows using `this & that`

Due to the lack of an explicit “drop rows” function in dplyr, many
people turn to `filter()`. But using `filter()` in this way often
requires replacing `,` separated conditions with `&` separated
conditions, wrapping the whole thing in parentheses, and then prefixing
it all with a `!`.

Take a look at this example. Our goal is:

> *Filter out* rows where the patient is deceased *and* the year of
> death was before 2012.

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

Compare that with this proposed usage of `filter_out()`:

``` r
patients |>
  filter_out(deceased, date < 2012)
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

This results in a `filter_out()` statement that precisely matches the
intent of the original problem statement. In other words, you can “write
it like you say it”.

In general, the following form of `filter()` can always be written as a
much simpler `filter_out()`:

``` r
data |> filter(!(this & that & those))
data |> filter_out(this, that, those)
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

> *Filter out* rows where the patient is deceased *and* the year of
> death was before 2012.

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
  mutate(to_keep = !(deceased & date < 2012)) |>
  mutate(what_filter_sees = to_keep & !is.na(to_keep))
```

    ## # A tibble: 7 × 5
    ##   name  deceased  date to_keep what_filter_sees
    ##   <chr> <lgl>    <dbl> <lgl>   <lgl>           
    ## 1 Anne  FALSE     2005 TRUE    TRUE            
    ## 2 Mark  TRUE      2010 FALSE   FALSE           
    ## 3 Sarah NA          NA NA      FALSE           
    ## 4 Davis TRUE      2020 TRUE    TRUE            
    ## 5 Max   NA        2010 NA      FALSE           
    ## 6 Derek FALSE       NA TRUE    TRUE            
    ## 7 Tina  TRUE        NA NA      FALSE

Because `filter()` treats `NA` as `FALSE`, we unexpectedly drop *more
than we expected*.

This phenomenon is rather confusing, but is due to the fact that
`filter()` is designed around the idea that you’re going to tell it
which rows to *keep*. In that case, ignoring `NA`s makes sense, i.e. if
you don’t *know* that you want to keep that row (because an `NA` is
ambiguous), then you probably don’t want to keep it.

This works well until you try to use `filter()` as a way to *filter out*
rows, at which point this behavior works against you. At this point,
most people reach for `is.na()` to help them out. Here’s what a
`filter()` call that only drops rows where you *know* the patient is
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
with that `coalesce()`. Here’s what using `filter_out()` would look
like:

``` r
patients |>
  filter_out(deceased, date < 2012)
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

Just like with `filter()`, `filter_out()` treats `NA` values as `FALSE`.
The difference is that `filter_out()` expects that you are going to tell
it which rows to *drop* (rather than which rows to keep), so the default
behavior of treating `NA` like `FALSE` works *with you* rather than
*against you*. It’s also much easier to understand when you look back on
it a year from now!

### Filtering out rows using `this | that`

Let’s look at another example. Our goal is:

> *Filter out* rows where class is “suv” *or* mpg is less than 15.

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

Because dplyr doesn’t have a way to drop rows, you’d reach for
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

> *Filter out* rows where class is “suv” *or* mpg is less than 15.

To achieve this we’ve had to:

- Flip the conditions (taking special care that `< 15` is now `>= 15`)
- Specially handle missing values
- Introduce `|`, at a minimum, or `!`, `()`, and `&` if not flipping the
  conditions

Here’s the `filter_out()` solution:

``` r
cars |>
  filter_out(class == "suv" | mpg < 15)
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

One beautiful thing about `filter_out()` is that any `|` separated
conditions can always be written as two sequential `filter_out()`
statements, meaning that you could also remove the `|` to simplify this
further as:

``` r
cars |>
  filter_out(class == "suv") |>
  filter_out(mpg < 15)
```

    ## # A tibble: 4 × 2
    ##   class   mpg
    ##   <chr> <dbl>
    ## 1 coupe    20
    ## 2 coupe    NA
    ## 3 <NA>     20
    ## 4 <NA>     NA

In general, the following form of `filter()` can always be written as a
much simpler `filter_out()`:

``` r
data |> filter(!(this | that | those))
data |> filter_out(this) |> filter_out(that) |> filter_out(those)
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

#### With `%in%`

`%in%` has historically been a common way to work around `NA` related
issues because it removes the `NA`s for you by performing an “exact
match” that only returns `TRUE` or `FALSE`, unlike `==` which propagates
missing values.

> *Filter out* rows where the country name is `"US"` *or* `"CA"`.

``` r
countries <- tibble(
  name = c("US", "CA", NA, "RU", "US", NA, "CA", "PR")
)

countries
```

    ## # A tibble: 8 × 1
    ##   name 
    ##   <chr>
    ## 1 US   
    ## 2 CA   
    ## 3 <NA> 
    ## 4 RU   
    ## 5 US   
    ## 6 <NA> 
    ## 7 CA   
    ## 8 PR

Directly translating the problem statement gives you
`name == "US" | name == "CA"`, and inverting that to work with filter
gives you:

``` r
countries |>
  filter(!(name == "US" | name == "CA"))
```

    ## # A tibble: 2 × 1
    ##   name 
    ##   <chr>
    ## 1 RU   
    ## 2 PR

But this is wrong because it once again drops your `NA`s. You’d need:

``` r
countries |>
  filter(!(name == "US" | name == "CA") | is.na(name))
```

    ## # A tibble: 4 × 1
    ##   name 
    ##   <chr>
    ## 1 <NA> 
    ## 2 RU   
    ## 3 <NA> 
    ## 4 PR

Savvy programmers might know that `%in%` can be used instead:

``` r
countries |>
  filter(!name %in% c("US", "CA"))
```

    ## # A tibble: 4 × 1
    ##   name 
    ##   <chr>
    ## 1 <NA> 
    ## 2 RU   
    ## 3 <NA> 
    ## 4 PR

This works because `%in%` preemptively turns `NA` into `FALSE` rather
than propagating them:

``` r
!(countries$name == "US" | countries$name == "CA")
```

    ## [1] FALSE FALSE    NA  TRUE FALSE    NA FALSE  TRUE

``` r
!countries$name %in% c("US", "CA")
```

    ## [1] FALSE FALSE  TRUE  TRUE FALSE  TRUE FALSE  TRUE

(Operator precedence also happens to work in your favor here, with `!`
being applied *after* the `%in%`, but many people would wrap
`name %in% c("US", "CA")` in an extra set of parentheses for clarity,
because that’s hard to remember!)

With `filter_out()`, your original translation of the problem would work
as expected:

``` r
countries |>
  filter_out(name == "US" | name == "CA")
```

    ## # A tibble: 4 × 1
    ##   name 
    ##   <chr>
    ## 1 <NA> 
    ## 2 RU   
    ## 3 <NA> 
    ## 4 PR

As mentioned earlier, this could be written as two sequential
`filter_out()` statements:

``` r
countries |>
  filter_out(name == "US") |>
  filter_out(name == "CA")
```

    ## # A tibble: 4 × 1
    ##   name 
    ##   <chr>
    ## 1 <NA> 
    ## 2 RU   
    ## 3 <NA> 
    ## 4 PR

And `%in%` works here as well, and doesn’t require the `!` out front:

``` r
countries |>
  filter_out(name %in% c("US", "CA"))
```

    ## # A tibble: 4 × 1
    ##   name 
    ##   <chr>
    ## 1 <NA> 
    ## 2 RU   
    ## 3 <NA> 
    ## 4 PR

#### `tidyr::drop_na()`

`tidyr::drop_na()` is a special case of `filter_out(this | that)`.
Because dplyr didn’t have a “drop rows” solution, years ago we added
`tidyr::drop_na()` as a way to drop rows where *any* specified column is
`NA`. The goal here is:

> *Filter out* rows where `deceased` *or* `date` are missing.

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

With `filter_out()`, you can express this as sequential `filter_out()`
calls if you just have 2-3 columns to work with:

``` r
patients |>
  filter_out(is.na(deceased)) |>
  filter_out(is.na(date))
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
patients |> filter_out(if_any(c(deceased, date), is.na))
```

    ## # A tibble: 3 × 3
    ##   name  deceased  date
    ##   <chr> <lgl>    <dbl>
    ## 1 Anne  FALSE     2005
    ## 2 Davis TRUE      2020
    ## 3 Derek FALSE     2000

### Filtering for rows using `this | that`

We’ve talked a lot about *dropping* rows, but this tidyup also proposes
a feature that helps with *keeping* rows when using conditions combined
with `|` - `when_any()`.

Our goal here is:

> *Filter for* rows where “US” and “CA” have a score between 200-300,
> *or* rows where “PR” and “RU” have a score between 100-200.

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
  filter(when_any(
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
`.when = c("all", "any")` style argument to `filter()` itself, which
feels less readable overall:

``` r
countries |>
  filter(
    name %in% c("US", "CA") & between(score, 200, 300),
    name %in% c("PR", "RU") & between(score, 100, 200),
    .when = "any"
  )
```

It’s also nice that `when_any()` and `when_all()` are useful outside of
just `filter()` and `filter_out()`, which we would not get with a
`.when` argument.

#### `when_all()`?

For completeness, `when_all()` also exists as a `pall()`
(“parallel-all”) style operator, but it isn’t as useful with `filter()`
or `filter_out()` because they already combine conditions with `&`. It’s
possible there will be cases when it can help you express complex
conditions combining `|` and `&` in a readable way, such as:

``` r
data |>
  filter(
    when_any(this, that),
    when_all(these, those)
  )
```

And it is also worth noting that these can be used outside of `filter()`
and `filter_out()`:

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

#### With `filter_out()`?

We’ve seen that `when_any()` can be useful with `filter()` because it
allows you to continue specifying your conditions separated by commas
rather than separated by `|` operators and wrapped in `()`. It’s worth
noting that `when_any()` does not add much value to `filter_out()`
because, as we saw in the previous section, `filter_out()` statements
involving `|` can be written as sequential `filter_out()` statements
instead, i.e.:

``` r
data |> filter_out(this | that)
data |> filter_out(this) |> filter_out(that)
```

This isn’t the case for `filter()` and `|`, hence the added value of
`when_any()`.

## Backwards compatibility

There are no breaking changes proposed in this tidyup.

## How to teach

`filter()` and `filter_out()` would be taught as a pair. Together:

- The presence of `filter_out()` should help imply that `filter()` is
  “filter for”

- `filter_out()` as an explicit way to drop rows eliminates the need for
  complex boolean logic

- `filter_out()` as an explicit way to drop rows results in intuitive
  missing value behavior

## Additional considerations

### SQL and dbplyr

`filter_out()` should be translatable to SQL using much of the same
infrastructure already used for `filter()`.

`when_all()` and `when_any()` should be translatable to SQL via explicit
usage of `|`, `&`, and `()`.

## Alternatives

### Alternate names for `filter()`

Because `filter()` is so ambiguous, a previous version of this tidyup
intended to alias it to a new function, `retain()`, as a very clear way
to “retain rows”. This would have been paired with `exclude()` as the
way to “exclude rows”. While these are clear, we worried about the idea
of aliasing `filter()` to a new verb. This would:

- Fracture the community into `filter()` users and `retain()` users

- Worry people about the future of `filter()`

- Ultimately not solve the problem, because programmers new to R will
  eventually run into legacy code, blog posts, or Stack Overflow
  questions that use `filter()`.

### Alternate names for `filter_out()`

- `exclude()`, as noted above, which would have been paired with
  `retain()`

- `reject()`

- `drop_rows()`

- `drop()`, but this is taken by base R

- `discard()`, but this is taken by purrr

Ultimately we decided that it made more sense to treat `filter_out()` as
a *variant* of the core verb `filter()`. This is similar to how
`slice_head()` and `slice_tail()` are *variants* of the core verb
`slice()`.

### `filter(.missing =)` and `filter_out(.missing =)`

An earlier version of this tidyup considered adding
`.missing = FALSE / TRUE` to `filter()` and `filter_out()`, with `FALSE`
being the default to “treat an `NA` like `FALSE`”. This was in response
to *many* requests for an argument like this on the dplyr Issues page.
After gathering more examples and feedback, we’ve determined:

- This argument is *highly* confusing to think about.

- The argument is a red herring. You actually wanted `filter_out()` all
  along.

Here’s the theoretical motivation for `.missing = TRUE`:

> *Filter out* rows where `x` and `y` are equal.

``` r
data <- tibble(
  x = c(1, 1, 1, 2, 2, 2, NA, NA, NA),
  y = c(1, 2, NA, 1, 2, NA, 1, 2, NA)
)

data
```

    ## # A tibble: 9 × 2
    ##       x     y
    ##   <dbl> <dbl>
    ## 1     1     1
    ## 2     1     2
    ## 3     1    NA
    ## 4     2     1
    ## 5     2     2
    ## 6     2    NA
    ## 7    NA     1
    ## 8    NA     2
    ## 9    NA    NA

Because dplyr didn’t have a “drop rows” style function, you’d reach for
`filter()` with `!=`:

``` r
data |> filter(x != y)
```

    ## # A tibble: 2 × 2
    ##       x     y
    ##   <dbl> <dbl>
    ## 1     1     2
    ## 2     2     1

But then you’d get frustrated that this didn’t drop *only* the rows
where `x` and `y` are equal, it also dropped the `NA` rows where the
result is ambiguous. So you’d add `is.na()` calls:

``` r
data |> filter(x != y | is.na(x) | is.na(y))
```

    ## # A tibble: 7 × 2
    ##       x     y
    ##   <dbl> <dbl>
    ## 1     1     2
    ## 2     1    NA
    ## 3     2     1
    ## 4     2    NA
    ## 5    NA     1
    ## 6    NA     2
    ## 7    NA    NA

At that point, people reasonably thought that a `.missing = TRUE`
argument might be useful. This would automatically treat `NA`s resulting
from `x != y` as `TRUE` rather than the default of `FALSE`. In other
words, they wanted to write:

``` r
data |> filter(x != y, .missing = TRUE)
```

But this is both a red herring and a fairly unintuitive bit of code to
come back and read a year from now.

We’ve determined that what we were *actually* missing was
`filter_out()`, because this is just:

``` r
data |> filter_out(x == y)
```

    ## # A tibble: 7 × 2
    ##       x     y
    ##   <dbl> <dbl>
    ## 1     1     2
    ## 2     1    NA
    ## 3     2     1
    ## 4     2    NA
    ## 5    NA     1
    ## 6    NA     2
    ## 7    NA    NA

This has the benefits of being short, intuitive, and clearly aligning
with the intent of the original goal.

Every issue / question below is actually a request for `filter_out()` in
disguise:

- [filter_out(col1 ==
  col2)](https://github.com/tidyverse/dplyr/issues/6432)
- [filter_out(Species ==
  “virginica”)](https://github.com/tidyverse/dplyr/issues/6013)
- [filter_out(y == “a”)](https://github.com/tidyverse/dplyr/issues/3196)
- [filter_out(col ==
  “str”)](https://stackoverflow.com/questions/46378437/how-to-filter-data-without-losing-na-rows-using-dplyr)
- [filter_out(var1 ==
  1)](https://stackoverflow.com/questions/32908589/why-does-dplyrs-filter-drop-na-values-from-a-factor-variable)

In the *extremely* rare cases where you might need `missing = TRUE`, you
can nest `when_all(na_rm = TRUE)` inside of `filter()` and
`filter_out()`. This propagates missings by default but `na_rm = TRUE`
removes missings from the computation. For an “all” style operation,
that is equivalent to treating missings like `TRUE` (i.e. `all()` and
`all(NA, na.rm = TRUE)` both return `TRUE`).

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

#### Intent vs Combine

<table style="width:98%;">
<colgroup>
<col style="width: 8%" />
<col style="width: 8%" />
<col style="width: 18%" />
<col style="width: 9%" />
<col style="width: 52%" />
</colgroup>
<thead>
<tr>
<th>Intent</th>
<th>Combine</th>
<th>Hypothetical usage %</th>
<th>Currently</th>
<th>Solution</th>
</tr>
</thead>
<tbody>
<tr>
<td>Keep</td>
<td>And</td>
<td>50%</td>
<td>✅</td>
<td><code>filter(a, b, c)</code></td>
</tr>
<tr>
<td>Keep</td>
<td>Or</td>
<td>5%</td>
<td>❌</td>
<td><code>filter(when_any(a, b, c))</code></td>
</tr>
<tr>
<td>Drop</td>
<td>And</td>
<td>35%</td>
<td>❌</td>
<td><code>filter_out(a, b, c)</code></td>
</tr>
<tr>
<td>Drop</td>
<td>Or</td>
<td>10%</td>
<td>❌</td>
<td><p><code>filter_out(when_any(a, b, c))</code></p>
<p>In practice:
<code>filter_out(a) |&gt; filter_out(b) |&gt; filter_out(c)</code></p></td>
</tr>
</tbody>
</table>

#### Intent vs Missings

| Intent | Missings | Outcome | Usefulness |
|----|----|----|----|
| Keep | Treat as `FALSE` | Keep rows where you *know* the conditions are `TRUE` | Very. Existing `filter()` behavior. |
| Drop | Treat as `FALSE` | Drop rows where you *know* the conditions are `TRUE` | Very. Simplifies “treat `filter()` as a drop” cases. |
| Keep | Treat as `TRUE` | Keep rows where conditions are `TRUE` or `NA` | Not. This is a `filter_out()` in disguise. |
| Drop | Treat as `TRUE` | Drop rows where conditions are `TRUE` or `NA` | Not. Never seen an example of this. |

#### Connection to vctrs

We purposefully don’t expose `missing` directly on the dplyr side. The
3-valued argument is quite complicated to think about. Instead it
bubbles up through both `filter()` / `filter_out()` using
`missing = FALSE` internally and `when_all()` / `when_any()`’s `na_rm`
argument.

Particularly confusing for the average consumer is that
`when_all(na_rm = TRUE)` maps to `list_pall(missing = TRUE)` but
`when_any(na_rm = TRUE)` maps to `list_pany(missing = FALSE)`. Exposing
only `na_rm = TRUE` saves users from having to do these very hard mental
gymnastics.

| vctrs | Data frame | Vector |
|----|----|----|
| `list_pall(missing = NULL)` |  | `when_all(na_rm = FALSE)` |
| `list_pall(missing = FALSE)` | `filter()` / `filter_out()` |  |
| `list_pall(missing = TRUE)` |  | `when_all(na_rm = TRUE)` |
| `list_pany(missing = NULL)` |  | `when_any(na_rm = FALSE)` |
| `list_pany(missing = FALSE)` |  | `when_any(na_rm = TRUE)` |
| `list_pany(missing = TRUE)` |  |  |

- `list_pall(missing = FALSE)`:

  - Interesting how this is useful as the `filter()` / `filter_out()`
    default behavior but becomes too confusing to try and expose in
    `when_all()` as `missing` vs the simpler `na_rm`. Keeping “the most
    flexible” vector function way in vctrs feels right since the
    `missing = FALSE` case here is less useful in a vector context. It
    doesn’t prevent you from doing `filter(when_all())` because the
    default propagates `NA` and then `filter()` itself does the
    `missing = FALSE` part.

- `list_pany(missing = TRUE)`:

  - Like `list_pall(missing = FALSE)`, this is not the useful variant to
    expose at the vector level. Also happens to not have an exposed data
    frame variant, so dplyr doesn’t expose it at all, which feels fine.
    Not a single example needed it.
