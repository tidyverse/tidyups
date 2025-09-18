
# Tidyup 7: Recoding and replacing values in the tidyverse

**Champion**: Davis

**Co-Champion**: Hadley

**Status**: Draft

``` r
library(dplyr, warn.conflicts = FALSE)
library(vctrs, warn.conflicts = FALSE)
```

## Abstract

Over the years, we have received numerous variations of the following
two questions:

- What is the best way to **recode** a column in the tidyverse?

- What is the best way to **replace** a few values within a column in
  the tidyverse?

The point of this tidyup is to argue that there is a hole in dplyr’s API
surrounding these two questions, and to propose a solution that fills
that hole by building on the intuition that people have around
`case_when()`’s formula interface.

Let’s first define exactly what we mean by recode and replace:

- **Recoding** *creates a new vector* with an entirely new type.
  Unmatched locations that aren’t handled by the recoding fall through
  to some default value, which uses a missing value if left unspecified.
  Output names are pulled from the names of the individual vectors that
  build the output values.

- **Replacing** *modifies an existing vector*, retaining its type.
  Unmatched locations that aren’t replaced retain the existing value of
  the original vector. Output names are left unchanged, regardless of
  whether the underlying value changed (mimicking `[<-` and
  `base::replace()`).

## Solution

In this tidyup, it’s most compelling to outline the solution first so we
can refer back to them when discussing motivation and showing examples.
The proposed solution is comprised of the following family of dplyr
functions:

``` r
# - `...` are for two sided formulas
# - `...` are mutually exclusive with `from/to` (interactive vs programmatic API)

# Recoding

case_when(
  ...,
  .default = NULL,
  .unmatched = c("default", "error"),
  .ptype = NULL,
  .size = NULL
)

recode_values(
  x,
  ...,
  from = NULL,
  to = NULL,
  default = NULL,
  unmatched = c("default", "error"),
  ptype = NULL
)

# Replacing

replace_when(
  x,
  ...
)

replace_values(
  x,
  ...,
  from = NULL,
  to = NULL
)
```

`case_match()` will be superseded in favor of the more powerful and
better named `recode_values()`.

For historical reasons, `case_when()` uses dotted argument names.
Ideally it would not, because the `...` of `case_when()` are not
intended to be named, though we technically allow this right now. We
correct this historical mistake with the other functions in the family.
We anticipate this will make them easier to teach.

Powering this family are the following lower level vctrs functions,
written in C for performance, and exposed in a package with minimal
dependencies for use in other R packages where dplyr is too heavy of a
dependency to take on. These are not the focus of this tidyup, but it is
worth knowing that they will also exist:

``` r
# - `...` must be empty, there is no formula API at this level

vec_case_when(
  cases,
  values,
  ...,
  default = NULL,
  unmatched = c("default", "error"),
  cases_arg = "cases",
  values_arg = "values",
  default_arg = "default",
  ptype = NULL,
  size = NULL,
  call = current_env()
)

vec_recode_values(
  x,
  ...,
  from,
  to,
  default = NULL,
  unmatched = c("default", "error"),
  from_as_list_of_vectors = FALSE,
  to_as_list_of_vectors = FALSE,
  x_arg = "x",
  from_arg = "from",
  to_arg = "to",
  default_arg = "default",
  ptype = NULL,
  call = current_env()
)

# Replacing

vec_replace_when(
  x,
  cases,
  values,
  ...,
  x_arg = "x",
  cases_arg = "cases",
  values_arg = "values",
  call = current_env()
)

vec_replace_values(
  x,
  ...,
  from,
  to,
  from_as_list_of_vectors = FALSE,
  to_as_list_of_vectors = FALSE,
  x_arg = "x",
  from_arg = "from",
  to_arg = "to",
  call = current_env()
)
```

It is straightforward to also construct `vec_if_else()` on top of
`vec_case_when()` and expose this extremely useful utility within a low
dependency package:

``` r
vec_if_else(
  condition,
  true,
  false,
  ...,
  missing = NULL,
  ptype = NULL,
  call = current_env()
)
```

## Implementation

Try for yourself at:

``` r
pak::pak("tidyverse/dplyr@feature/case-family")
```

Note that this implementation is not finished yet, and may have bugs.

- [vctrs PR](https://github.com/r-lib/vctrs/pull/1984)

- [dplyr
  branch](https://github.com/tidyverse/dplyr/tree/feature/case-family)

## Motivation

While dplyr’s `case_when()` and `case_match()` are widely loved, over
the years we have learned that there are still holes in the API that
they are trying to fill:

- They are designed to *create* new vectors, making them well suited for
  recoding, but poorly suited for replacing where you are *modifying* an
  existing vector.

- They lack *programmatic* interfaces, like `plyr::mapvalues()` or
  `dplyr::recode(x, !!!cases)`. This is immediately noticeable when
  using a lookup table with `case_match()`. There is no intuitive way to
  define the lookup table at the top of a script and pass it through to
  the `case_match()` call, which we would encourage as a best practice
  if it was possible.

- `case_match()` unfortunately has an unintuitive name. It tries to lean
  into the SQL “simple” `CASE WHEN` analogy combined with the fact that
  it works similarly to a `match()` call, but this doesn’t land for most
  users, and many don’t know it even exists. `recode_values()` and
  `replace_values()` are much more evocative names.

Below, we explore examples of both recoding and replacing (many
extracted from real GitHub issues or discussions with users), comparing
existing solutions to the proposed API.

### Recoding

#### As an inline lookup table with `case_match()`

Consider the following Likert scale scores. We’d like to recode these
from their numeric values to their character counterparts.

``` r
data <- tibble(
  score = c(1, 2, 3, 4, 5, 2, 3, 1, 4)
)
```

To do this with existing tools, you’ll likely be inclined to reach for
`case_when()` or `case_match()`:

``` r
data |>
  mutate(
    score = case_when(
      score == 1 ~ "Strongly disagree",
      score == 2 ~ "Disagree",
      score == 3 ~ "Neutral",
      score == 4 ~ "Agree",
      score == 5 ~ "Strongly agree"
    )
  )
```

    ## # A tibble: 9 × 1
    ##   score            
    ##   <chr>            
    ## 1 Strongly disagree
    ## 2 Disagree         
    ## 3 Neutral          
    ## 4 Agree            
    ## 5 Strongly agree   
    ## 6 Disagree         
    ## 7 Neutral          
    ## 8 Strongly disagree
    ## 9 Agree

``` r
# Or
data |>
  mutate(
    score = case_match(
      score,
      1 ~ "Strongly disagree",
      2 ~ "Disagree",
      3 ~ "Neutral",
      4 ~ "Agree",
      5 ~ "Strongly agree"
    )
  )
```

    ## # A tibble: 9 × 1
    ##   score            
    ##   <chr>            
    ## 1 Strongly disagree
    ## 2 Disagree         
    ## 3 Neutral          
    ## 4 Agree            
    ## 5 Strongly agree   
    ## 6 Disagree         
    ## 7 Neutral          
    ## 8 Strongly disagree
    ## 9 Agree

This is okay in some cases, but for a fixed Likert scale like this, it’s
typically more readable to extract the mapping out into a separate
lookup table and refer to that within the `mutate()` call.
Unfortunately, `case_when()` and `case_match()` don’t provide an easy
way to do this, so you are forced to “inline” the lookup table. With the
newly proposed `recode_values()`, you can use `from` and `to` instead,
which work similarly to `plyr::mapvalues()`:

``` r
# Likert scale lookup table
# fmt: skip
likert_lookup <- tribble(
  ~from, ~to,
  1, "Strongly disagree",
  2, "Disagree",
  3, "Neutral",
  4, "Agree",
  5, "Strongly agree"
)

data |>
  mutate(
    score = recode_values(
      score,
      from = pull(likert_lookup, from),
      to = pull(likert_lookup, to)
    )
  )
```

    ## # A tibble: 9 × 1
    ##   score            
    ##   <chr>            
    ## 1 Strongly disagree
    ## 2 Disagree         
    ## 3 Neutral          
    ## 4 Agree            
    ## 5 Strongly agree   
    ## 6 Disagree         
    ## 7 Neutral          
    ## 8 Strongly disagree
    ## 9 Agree

It’s quite difficult to achieve something similar with `case_match()`.
It requires a somewhat advanced level of formula manipulation for a
fairly simple task:

``` r
cases <- purrr::map2(likert_lookup$from, likert_lookup$to, \(from, to) {
  rlang::new_formula(from, to)
})

data |>
  mutate(score = case_match(score, !!!cases))
```

    ## # A tibble: 9 × 1
    ##   score            
    ##   <chr>            
    ## 1 Strongly disagree
    ## 2 Disagree         
    ## 3 Neutral          
    ## 4 Agree            
    ## 5 Strongly agree   
    ## 6 Disagree         
    ## 7 Neutral          
    ## 8 Strongly disagree
    ## 9 Agree

It’s also common for a lookup table like `likert_lookup` to actually be
defined in a separate CSV file, making it very awkward to inline it. The
`from/to` API makes this much easier:

``` r
likert_lookup <- read_csv("lookup.csv")

data |>
  mutate(
    score = recode_values(
      score,
      from = pull(likert_lookup, from),
      to = pull(likert_lookup, to)
    )
  )
```

Note that `recode_values()` still supports this “inline” lookup table
via a formula API that is mutually exclusive with the `from` / `to` API.

``` r
data |>
  mutate(
    score = recode_values(
      score,
      1 ~ "Strongly disagree",
      2 ~ "Disagree",
      3 ~ "Neutral",
      4 ~ "Agree",
      5 ~ "Strongly agree"
    )
  )
```

    ## # A tibble: 9 × 1
    ##   score            
    ##   <chr>            
    ## 1 Strongly disagree
    ## 2 Disagree         
    ## 3 Neutral          
    ## 4 Agree            
    ## 5 Strongly agree   
    ## 6 Disagree         
    ## 7 Neutral          
    ## 8 Strongly disagree
    ## 9 Agree

With the formula API, `recode_values()` is an exact replacement for
`case_match()`, but comes with a powerful programmatic API as well.

#### As a lookup table with a join

More advanced users might skip `case_when()` and `case_match()` in favor
of a join. This isn’t wrong, but it isn’t an intuitive operation to
reach for as a beginner, and typically requires extra steps if you want
the resulting column to have the same name as the original one.

``` r
# Likert scale
# fmt: skip
likert_lookup <- tribble(
  ~from, ~to,
  1, "Strongly disagree",
  2, "Disagree",
  3, "Neutral",
  4, "Agree",
  5, "Strongly agree"
)

# If you are okay with having both columns in the result, you stop here
left_join(data, likert_lookup, join_by(score == from))
```

    ## # A tibble: 9 × 2
    ##   score to               
    ##   <dbl> <chr>            
    ## 1     1 Strongly disagree
    ## 2     2 Disagree         
    ## 3     3 Neutral          
    ## 4     4 Agree            
    ## 5     5 Strongly agree   
    ## 6     2 Disagree         
    ## 7     3 Neutral          
    ## 8     1 Strongly disagree
    ## 9     4 Agree

``` r
# But often you want this
left_join(data, likert_lookup, join_by(score == from)) |>
  select(-score) |>
  rename(score = to)
```

    ## # A tibble: 9 × 1
    ##   score            
    ##   <chr>            
    ## 1 Strongly disagree
    ## 2 Disagree         
    ## 3 Neutral          
    ## 4 Agree            
    ## 5 Strongly agree   
    ## 6 Disagree         
    ## 7 Neutral          
    ## 8 Strongly disagree
    ## 9 Agree

With `recode_values()`, you recode directly within a `mutate()` call,
which feels intuitive, and can assign directly back to the name of the
column you recoded.

``` r
data |>
  mutate(
    score = recode_values(
      score,
      from = pull(likert_lookup, from),
      to = pull(likert_lookup, to)
    )
  )
```

    ## # A tibble: 9 × 1
    ##   score            
    ##   <chr>            
    ## 1 Strongly disagree
    ## 2 Disagree         
    ## 3 Neutral          
    ## 4 Agree            
    ## 5 Strongly agree   
    ## 6 Disagree         
    ## 7 Neutral          
    ## 8 Strongly disagree
    ## 9 Agree

It’s also worth mentioning that a join requires *data frames* as inputs,
but recoding is really a *vector* level operation. Note how
`recode_values()` doesn’t require data frames at all, making it suitable
for package level programmatic usage as well.

``` r
score <- data$score
from <- likert_lookup$from
to <- likert_lookup$to

recode_values(score, from = from, to = to)
```

    ## [1] "Strongly disagree" "Disagree"          "Neutral"          
    ## [4] "Agree"             "Strongly agree"    "Disagree"         
    ## [7] "Neutral"           "Strongly disagree" "Agree"

``` r
# In this case you may consider using the vctrs variant as well, depending on
# your use case
vec_recode_values(score, from = from, to = to)
```

    ## [1] "Strongly disagree" "Disagree"          "Neutral"          
    ## [4] "Agree"             "Strongly agree"    "Disagree"         
    ## [7] "Neutral"           "Strongly disagree" "Agree"

#### Accidentally dropping a value

When recoding a vector, it’s entirely possible that you might
accidentally miss a value. This results in it being silently converted
to a missing value. Doubly confusing is that this missing value looks
the same as preexisting missing values:

``` r
data <- tibble(
  score = c(0, 1, 2, NA, 4, 5, 2, 3, 1, 4)
)

# Missed the `0`
data |>
  mutate(
    score_recoded = case_match(
      score,
      1 ~ "Strongly disagree",
      2 ~ "Disagree",
      3 ~ "Neutral",
      4 ~ "Agree",
      5 ~ "Strongly agree"
    )
  )
```

    ## # A tibble: 10 × 2
    ##    score score_recoded    
    ##    <dbl> <chr>            
    ##  1     0 <NA>             
    ##  2     1 Strongly disagree
    ##  3     2 Disagree         
    ##  4    NA <NA>             
    ##  5     4 Agree            
    ##  6     5 Strongly agree   
    ##  7     2 Disagree         
    ##  8     3 Neutral          
    ##  9     1 Strongly disagree
    ## 10     4 Agree

Some people guard against this by inserting and detecting a special
`.default` value:

``` r
data <- data |>
  mutate(
    score_recoded = case_match(
      score,
      1 ~ "Strongly disagree",
      2 ~ "Disagree",
      3 ~ "Neutral",
      4 ~ "Agree",
      5 ~ "Strongly agree",
      NA ~ NA,
      .default = "MISSED_ME"
    )
  )

if (any(data$score_recoded == "MISSED_ME")) {
  stop("Oh no!")
}
```

    ## Error: Oh no!

With the proposal in this tidyup, both `case_when()` and
`recode_values()` would gain an `unmatched` argument which can
optionally error on unmatched values rather than falling through to a
`default`. Note that you have to explicitly handle every value, even
missing values, so they need to be included in your lookup table if you
are using one.

``` r
# Likert scale
# fmt: skip
likert_lookup <- tribble(
  ~from, ~to,
  1, "Strongly disagree",
  2, "Disagree",
  3, "Neutral",
  4, "Agree",
  5, "Strongly agree"
)

likert_lookup <- add_row(likert_lookup, from = NA, to = NA)

# Errors on the `0` that otherwise would fall through to `default`
data |>
  mutate(
    score = recode_values(
      score,
      from = pull(likert_lookup, from),
      to = pull(likert_lookup, to),
      unmatched = "error"
    )
  )
```

    ## Error in `mutate()`:
    ## ℹ In argument: `score = recode_values(...)`.
    ## Caused by error in `recode_values()`:
    ## ! Each output location must be matched.
    ## ✖ Location 1 is unmatched.

This is similar to using the join argument of the same name, but that
requires a bit of mental gymnastics to get the right combination of
`*_join()` and `unmatched` for this particular use case. In this case,
it’s an `inner_join()` with `unmatched = c("error", "drop")` to declare
that you’re okay with unmatched keys from `y` dropping out, but you
don’t want any unmatched `x` keys.

``` r
inner_join(
  data,
  likert_lookup,
  join_by(score == from),
  unmatched = c("error", "drop")
)
```

    ## Error in `inner_join()`:
    ## ! Each row of `x` must have a match in `y`.
    ## ℹ Row 1 of `x` does not have a match.

#### References

Questions and discussions related to recoding values which were
referenced when creating this tidyup:

- <a
  href="https://bsky.app/profile/randvegan.bsky.social/post/3lsab7xfb6s2x"
  class="uri">Bsky conversation about this</a>
- <a
  href="https://www.linkedin.com/posts/libbyheeren_rstats-activity-7343291858275487744-XlPl?utm_source=share&amp;utm_medium=member_desktop&amp;rcm=ACoAAAy7IywB2qfaREGGoCca5XkthJ2hLjru6ts"
  class="uri">Libby’s Linkedin post about this</a>
- <a href="https://github.com/tidyverse/dplyr/issues/7694"
  class="uri">Michael Chirico being confused about this</a>
- [A request for \`plyr::mapvalues()\`, which \`case_match()\` doesn’t
  replicate](https://github.com/tidyverse/dplyr/issues/7027)
- [Another request for \`plyr::mapvalues()\`, which \`case_match()\`
  doesn’t
  replicate](https://github.com/tidyverse/dplyr/issues/5919#issuecomment-1943124726)
- [Request for from/to style documentation examples pulled from
  recode()](https://github.com/tidyverse/dplyr/issues/6623)
- [Me realizing that even \`vec_case_match()\` isn’t perfect to replace
  \`mapvalues()\`](https://github.com/r-lib/vctrs/issues/1622)
- [{funs} request for
  mapvalues()](https://github.com/tidyverse/funs/issues/15)
- [{funs} issue about Lionel’s vec_recode()
  attempt](https://github.com/tidyverse/funs/issues/29)
- [coolbutuseless strict
  case_when](https://coolbutuseless.github.io/2018/09/06/strict-case_when/)

### Replacing

#### With `.default = col`

Traditionally, to replace a few values within a column in the tidyverse
you’d use either `case_when()` or `case_match()` and set
`.default = col` to retain the original vector’s values in locations
that you weren’t replacing.

``` r
# Collapse some, but not all, of these school names into common buckets
schools <- tibble(
  name = c(
    "UNC",
    "Chapel Hill",
    NA,
    "Duke",
    "Duke University",
    "UNC",
    "NC State",
    "ECU"
  )
)

schools |>
  mutate(
    name = case_match(
      name,
      c("UNC", "Chapel Hill") ~ "UNC Chapel Hill",
      c("Duke", "Duke University") ~ "Duke",
      .default = name
    )
  )
```

    ## # A tibble: 8 × 1
    ##   name           
    ##   <chr>          
    ## 1 UNC Chapel Hill
    ## 2 UNC Chapel Hill
    ## 3 <NA>           
    ## 4 Duke           
    ## 5 Duke           
    ## 6 UNC Chapel Hill
    ## 7 NC State       
    ## 8 ECU

[This operation is so
common](https://github.com/tidyverse/dplyr/issues/7696) that it feels
like it deserves its own name that doesn’t require you to set
`.default = name`. For that, this tidyup proposes `replace_values()`:

``` r
schools |>
  mutate(
    name = replace_values(
      name,
      c("UNC", "Chapel Hill") ~ "UNC Chapel Hill",
      c("Duke", "Duke University") ~ "Duke"
    )
  )
```

    ## # A tibble: 8 × 1
    ##   name           
    ##   <chr>          
    ## 1 UNC Chapel Hill
    ## 2 UNC Chapel Hill
    ## 3 <NA>           
    ## 4 Duke           
    ## 5 Duke           
    ## 6 UNC Chapel Hill
    ## 7 NC State       
    ## 8 ECU

Like `recode_values()`, `replace_values()` has an alternative `from` and
`to` API that works well with lookup tables and allows you to move your
replacement mapping up to the top of your script.

``` r
# fmt: skip
schools_lookup <- tribble(
  ~from, ~to,
  "UNC", "UNC Chapel Hill",
  "Chapel Hill", "UNC Chapel Hill",
  "Duke", "Duke",
  "Duke University", "Duke"
)

schools |>
  mutate(
    name = replace_values(
      name,
      from = pull(schools_lookup, from),
      to = pull(schools_lookup, to)
    )
  )
```

    ## # A tibble: 8 × 1
    ##   name           
    ##   <chr>          
    ## 1 UNC Chapel Hill
    ## 2 UNC Chapel Hill
    ## 3 <NA>           
    ## 4 Duke           
    ## 5 Duke           
    ## 6 UNC Chapel Hill
    ## 7 NC State       
    ## 8 ECU

An extremely neat feature of the `from` and `to` API is that they also
take *lists* of vectors that describe the mapping, which has been
designed to work elegantly with the fact that `tribble()` can create
list columns, allowing you to further collapse this lookup table:

``` r
# Condensed lookup table with a `many:1` mapping per row
# fmt: skip
schools_lookup <- tribble(
  ~from, ~to,
  c("UNC", "Chapel Hill"), "UNC Chapel Hill",
  c("Duke", "Duke University"), "Duke"
)

# Note the type of `from` here
schools_lookup
```

    ## # A tibble: 2 × 2
    ##   from      to             
    ##   <list>    <chr>          
    ## 1 <chr [2]> UNC Chapel Hill
    ## 2 <chr [2]> Duke

``` r
schools_lookup$from
```

    ## [[1]]
    ## [1] "UNC"         "Chapel Hill"
    ## 
    ## [[2]]
    ## [1] "Duke"            "Duke University"

``` r
# Works the same as before
schools |>
  mutate(
    name = replace_values(
      name,
      from = pull(schools_lookup, from),
      to = pull(schools_lookup, to)
    )
  )
```

    ## # A tibble: 8 × 1
    ##   name           
    ##   <chr>          
    ## 1 UNC Chapel Hill
    ## 2 UNC Chapel Hill
    ## 3 <NA>           
    ## 4 Duke           
    ## 5 Duke           
    ## 6 UNC Chapel Hill
    ## 7 NC State       
    ## 8 ECU

#### With the native pipe

Using `case_match(x, .default = x)` has [an annoying
issue](https://github.com/tidyverse/dplyr/issues/6962) when the native
pipe is involved:

``` r
# Let's assume we have a little preprocessing to do before we can use
# `case_match()`. We need to strip off the `Program-` prefix for some
# of the schools.

schools <- tibble(
  name = c(
    "Program-UNC",
    "Chapel Hill",
    NA,
    "Duke",
    "Duke University",
    "UNC",
    "Program-NC State",
    "ECU"
  )
)

# Pipe placeholder can only appear once, so this is invalid R code
schools |>
  mutate(
    name = name |>
      stringr::str_remove("Program-") |>
      case_match(
        .x = _,
        c("UNC", "Chapel Hill") ~ "UNC Chapel Hill",
        c("Duke", "Duke University") ~ "Duke",
        .default = _
      )
  )
```

    ## Error in case_match(.x = "_", c("UNC", "Chapel Hill") ~ "UNC Chapel Hill", : pipe placeholder may only appear once (<input>:23:7)

Because the pipe placeholder of `_` can only appear once per function
call, this is invalid R code! `replace_values()` happens to solve this
problem elegantly by only requiring you to specify the column one time.
In fact, because the pipe placeholder is automatically assigned to the
first input, you can remove the `_` altogether.

``` r
schools |>
  mutate(
    name = name |>
      stringr::str_remove("Program-") |>
      replace_values(
        c("UNC", "Chapel Hill") ~ "UNC Chapel Hill",
        c("Duke", "Duke University") ~ "Duke"
      )
  )
```

    ## # A tibble: 8 × 1
    ##   name           
    ##   <chr>          
    ## 1 UNC Chapel Hill
    ## 2 UNC Chapel Hill
    ## 3 <NA>           
    ## 4 Duke           
    ## 5 Duke           
    ## 6 UNC Chapel Hill
    ## 7 NC State       
    ## 8 ECU

#### Type stability

One subtle issue with using `case_when(.default = x)` or
`case_match(.default = x)` is that it isn’t type stable on `x`, which
you probably do want when you’re just replacing a few values.

``` r
# Pretend it is important for this to be an <integer> column
x <- c(1L, 2L, 3L)

# Replacing `1` with `0` and retaining `x` as the default results in a <double>
# vector, because the common type of `x` (integer) and `0` (double) is double.
# We've lost the original <integer> type.
typeof(case_match(x, 1 ~ 0, .default = x))
```

    ## [1] "double"

Remember that `case_when()` and `case_match()` were originally designed
to *create totally new vectors*, so they aren’t meant to be type stable
on their inputs like you might expect here. You can force type
stability, but it’s a bit of work:

``` r
typeof(case_match(x, 1 ~ 0, .default = x, .ptype = x))
```

    ## [1] "integer"

`replace_values()` is an all-around better solution here, it’s type
stable by default, and much shorter!

``` r
typeof(replace_values(x, 1 ~ 0))
```

    ## [1] "integer"

This is particularly useful when partially recoding factors to existing
levels.

``` r
pets <- tibble(
  name = c("Max", "Bella", "Chuck", "Luna", "Cooper"),
  # Note the so-far unused "puppy" level:
  type = factor(
    c("dog", "dog", "cat", "dog", "cat"),
    levels = c("dog", "cat", "puppy")
  ),
  age = c(1, 3, 5, 2, 4)
)

# Recode some values to `"puppy"` using a character on the RHS
pets |>
  mutate(
    type = type |>
      replace_when(type == "dog" & age <= 2 ~ "puppy")
  )
```

    ## # A tibble: 5 × 3
    ##   name   type    age
    ##   <chr>  <fct> <dbl>
    ## 1 Max    puppy     1
    ## 2 Bella  dog       3
    ## 3 Chuck  cat       5
    ## 4 Luna   puppy     2
    ## 5 Cooper cat       4

``` r
# Note the type safety! Only existing levels can be used here.
pets |>
  mutate(
    type = type |>
      replace_when(type == "dog" & age <= 2 ~ "pup")
  )
```

    ## Error in `mutate()`:
    ## ℹ In argument: `type = replace_when(type, type == "dog" & age <= 2 ~
    ##   "pup")`.
    ## Caused by error in `replace_when()`:
    ## ! Can't convert from `..1 (right)` <character> to <factor<00b23>> due to loss of generality.
    ## • Locations: 1, 2, 3, 4, 5

The corresponding call to `case_when()` is much less intuitive due to
the fact that the common type of character and factor is character,
which means that the output of this `case_when()` call would be a
character without the `.ptype` argument to force it to be a factor.

``` r
pets |>
  mutate(
    type = case_when(
      type == "dog" & age <= 2 ~ "puppy",
      .default = type,
      .ptype = type
    )
  )
```

    ## # A tibble: 5 × 3
    ##   name   type    age
    ##   <chr>  <fct> <dbl>
    ## 1 Max    puppy     1
    ## 2 Bella  dog       3
    ## 3 Chuck  cat       5
    ## 4 Luna   puppy     2
    ## 5 Cooper cat       4

#### Retaining names

Another subtle difference between *recoding* and *replacing* is where
the output names are pulled from. With recoding, you create an entirely
new vector, so output names are built from the input vectors that end up
building the result:

``` r
x <- c(a = 1, b = 2, c = 3)

from <- c(2, 3)
to <- c(x = 20, y = 30)

recode_values(x, from = from, to = to)
```

    ##     x  y 
    ## NA 20 30

With replacing, you modify an *existing* vector, so output names are the
exact same as the original vector, regardless of whether you modified
the underlying value or not.

``` r
replace_values(x, from = from, to = to)
```

    ##  a  b  c 
    ##  1 20 30

This is the exact principle used by `base::replace()` and `[<-`:

``` r
# Notice how `b` stays even though `20` takes the place of `2`
replace(x, 2, c(x = 20))
```

    ##  a  b  c 
    ##  1 20  3

#### Retaining intent

When you’re replacing a few values in one column based on a condition
computed from another, you typically reach for `case_when()`. However,
because you provide the column as the last argument (the `.default`),
the intent of the operation is lost. With `replace_when()`, you specify
the column as the first argument, which helps you retain a natural
reading order and matches a common tidyverse pattern where the first
input usually correlates strongly to the type, size, and values of the
output. It also works nicely with the pipe!

``` r
# Svalbard and Jan Mayen technically have the same ISO code, but for our
# analysis assume we need them to be different
data <- tibble(
  country = c("USA", "Svalbard", "Jan Mayen", "Canada"),
  iso_code = c("US", "SJ", "SJ", "CA")
)

# The intent to just replace a few values in `iso_code` gets lost
# due to the fact that it is specified as the last argument
data |>
  mutate(
    iso_code = case_when(
      country == "Svalbard" ~ "SJ-Svalbard",
      country == "Jan Mayen" ~ "SJ-Jan Mayen",
      .default = iso_code
    )
  )
```

    ## # A tibble: 4 × 2
    ##   country   iso_code    
    ##   <chr>     <chr>       
    ## 1 USA       US          
    ## 2 Svalbard  SJ-Svalbard 
    ## 3 Jan Mayen SJ-Jan Mayen
    ## 4 Canada    CA

``` r
# `replace_when()` allows you to pull it to the front, or even pipe it in!
# This makes the intent of your code much clearer.
data |>
  mutate(
    iso_code = iso_code |>
      replace_when(
        country == "Svalbard" ~ "SJ-Svalbard",
        country == "Jan Mayen" ~ "SJ-Jan Mayen"
      )
  )
```

    ## # A tibble: 4 × 2
    ##   country   iso_code    
    ##   <chr>     <chr>       
    ## 1 USA       US          
    ## 2 Svalbard  SJ-Svalbard 
    ## 3 Jan Mayen SJ-Jan Mayen
    ## 4 Canada    CA

#### `na_if()` alternative

A fun result that falls out of `replace_values()` is that it’s a more
flexible (and likely more intuitive) alternative to `na_if()`:

``` r
x <- c(1, 2, 0, -99, 12)

# To convert `0` and `-99` to `NA`, you have to do it in two calls
x |> na_if(0) |> na_if(-99)
```

    ## [1]  1  2 NA NA 12

``` r
x |> replace_values(from = c(0, -99), to = NA)
```

    ## [1]  1  2 NA NA 12

#### `coalesce()` alternative

It’s also interesting to think of `replace_values()` as a simple
`coalesce()` alternative:

``` r
x <- c(1, 2, NA, 3, NA, 5)

coalesce(x, 0)
```

    ## [1] 1 2 0 3 0 5

``` r
replace_values(x, NA ~ 0)
```

    ## [1] 1 2 0 3 0 5

#### `tidyr::replace_na()` alternative

`replace_values()` is also a generalization of `tidyr::replace_na()`:

``` r
data <- tibble(
  x = c(1, 2, NA, 4, NA),
  y = c(NA, "a", NA, "b", NA)
)

tidyr::replace_na(data, list(x = 0, y = "unknown"))
```

    ## # A tibble: 5 × 2
    ##       x y      
    ##   <dbl> <chr>  
    ## 1     1 unknown
    ## 2     2 a      
    ## 3     0 unknown
    ## 4     4 b      
    ## 5     0 unknown

``` r
data |>
  mutate(
    x = tidyr::replace_na(x, 0),
    y = tidyr::replace_na(y, "unknown")
  )
```

    ## # A tibble: 5 × 2
    ##       x y      
    ##   <dbl> <chr>  
    ## 1     1 unknown
    ## 2     2 a      
    ## 3     0 unknown
    ## 4     4 b      
    ## 5     0 unknown

``` r
data |>
  mutate(
    x = replace_values(x, NA ~ 0),
    y = replace_values(y, NA ~ "unknown")
  )
```

    ## # A tibble: 5 × 2
    ##       x y      
    ##   <dbl> <chr>  
    ## 1     1 unknown
    ## 2     2 a      
    ## 3     0 unknown
    ## 4     4 b      
    ## 5     0 unknown

A similar comparison can be done for `naniar::replace_na_with()` and
`naniar::replace_with_na()`.

#### Multiple columns

In some cases, you may need to replace values in multiple columns at
once. In our experience, this isn’t quite as common as the other
examples, but it’s still possible using the fact that `replace_values()`
and friends can take data frames as inputs, and that `mutate()` will
automatically unpack unnamed data frames:

``` r
data <- tibble(
  x = c(2, 5, 12, 15, 18, 3, 3, 7),
  y = c(10, 5, 7, 1, 6, 4, 3, 9),
  age = c(20, 15, 25, 45, 38, 41, 45, 46)
)

data |>
  mutate(replace_when(
    pick(x, y),
    age < 20 ~ tibble(x = 0, y = 0),
    age > 40 ~ tibble(x = NA, y = NA)
  ))
```

    ## # A tibble: 8 × 3
    ##       x     y   age
    ##   <dbl> <dbl> <dbl>
    ## 1     2    10    20
    ## 2     0     0    15
    ## 3    12     7    25
    ## 4    NA    NA    45
    ## 5    18     6    38
    ## 6    NA    NA    41
    ## 7    NA    NA    45
    ## 8    NA    NA    46

#### References

Questions and discussions related to replacing values which were
referenced when creating this tidyup:

- <a href="https://github.com/tidyverse/dplyr/issues/4050"
  class="uri">2018 - Conditionally mutate selected rows, our 2nd oldest
  open issue</a>
- [{funs} attempt at
  recode_when()](https://github.com/tidyverse/funs/pull/66)
- [Updating multiple values at once in
  \`na_if()\`](https://github.com/tidyverse/dplyr/issues/7651)
- [A request for
  “case_replace()”](https://github.com/tidyverse/dplyr/issues/7696)
- [\`case_match(.x, .default = .x)\` doesn’t play nicely with pipes!
  Underrated problem
  IMO.](https://github.com/tidyverse/dplyr/issues/6962)
- <a href="https://x.com/antoine_fabri/status/1392127389195452416"
  class="uri">mutate_where() twitter request</a>

## Backwards compatibility

### `case_match()`

`case_match()` will be superseded in favor of `recode_values()`, a more
powerful and better named alternative. All existing code using
`case_match()` will continue to work.

## How to teach

When teaching these new functions, it’s probably best to introduce them
in pairs, i.e. `case_when() / replace_when()` and
`recode_values() / replace_values()`, but within the same overarching
lesson.

One key thing to avoid is talking about `dplyr::recode()`. It is
unfortunate that this function shares such a similar name as
`recode_values()`, but `dplyr::recode()` has been superseded for a while
now, and the API of `recode_values()` is much better. In particular, the
pattern of splicing in a named vector to `dplyr::recode()` using `!!!`
is now much more elegantly handled using `from` and `to` arguments of
`recode_values()`.

``` r
set.seed(1234)

x <- sample(c("a", "b", "c"), 6, replace = TRUE)

lookup <- c(
  a = "Apple",
  b = "Banana",
  c = "Candy"
)

dplyr::recode(x, !!!lookup)
```

    ## [1] "Banana" "Banana" "Apple"  "Candy"  "Apple"  "Apple"

``` r
lookup <- tibble::enframe(lookup, "from", "to")

dplyr::recode_values(x, from = lookup$from, to = lookup$to)
```

    ## [1] "Banana" "Banana" "Apple"  "Candy"  "Apple"  "Apple"

## Additional considerations

### dbplyr

`case_when()` already nicely translates to SQL’s “searched” `CASE WHEN`,
and `case_match()` already translates to SQL’s “simple” `CASE WHEN`.
`recode_values()` supersedes `case_match()` with a similar API, so we
anticipate that it should also translate elegantly to a “simple”
`CASE WHEN` statement.

Both `replace_when()` and `replace_values()` could be treated as
`case_when()` and `recode_values()` calls where you already know what
the fallthrough `ELSE` condition is - the original column. This wouldn’t
be type stable on the original column type though, so we may also want
to look into translating to `UPDATE + SET + WHERE` alongside the simple
`CASE WHEN`, like [this
example](https://stackoverflow.com/questions/15766102/i-want-to-use-case-statement-to-update-some-records-in-sql-server-2005).

The `from` and `to` API of `recode_values()` and `replace_values()`
could likely be programmatically translated to their equivalent formula
API, and run back through the existing code that handles the formula API
for these functions.

It is unlikely that the new `unmatched` argument could be supported.

## Appendix

### References

Sources of inspiration considered while designing these APIs:

- [naniar](https://naniar.njtierney.com/)
- [tidyr::replace_na()](https://tidyr.tidyverse.org/reference/replace_na.html)
- [base::replace()](https://stat.ethz.ch/R-manual/R-devel/library/base/html/replace.html)
- [plyr::mapvalues()](https://cran.r-project.org/web/packages/plyr/refman/plyr.html#mapvalues)
- [kit::vswitch()](https://cran.r-project.org/web/packages/kit/refman/kit.html#vswitch+2Fnswitch)
- [Excel’s
  XLOOKUP](https://support.microsoft.com/en-us/office/xlookup-function-b7fd680e-6d10-43e6-84f9-88eae8bf5929)
- [Stata’s recode](https://www.stata.com/manuals/drecode.pdf)

### Full API

While working on this tidyup, we developed the following table of
possible ways you might want to use these functions to ensure that we
captured every possibility we could think of.

<table style="width:97%;">
<colgroup>
<col style="width: 9%" />
<col style="width: 6%" />
<col style="width: 6%" />
<col style="width: 3%" />
<col style="width: 43%" />
<col style="width: 27%" />
</colgroup>
<thead>
<tr class="header">
<th>Usage</th>
<th>Action</th>
<th>Style</th>
<th>RHS</th>
<th>Function</th>
<th>Notes</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>Interactive</td>
<td>Create</td>
<td>if/else</td>
<td>1</td>
<td>case_when(condition ~ “value”, condition2 ~ “value2”)</td>
<td>Exists</td>
</tr>
<tr class="even">
<td>Interactive</td>
<td>Create</td>
<td>if/else</td>
<td>N</td>
<td>case_when(condition ~ column, condition2 ~ column2)</td>
<td>Exists</td>
</tr>
<tr class="odd">
<td>Interactive</td>
<td>Create</td>
<td>switch</td>
<td>1</td>
<td>recode_values(x, 1 ~ “value1”, 2 ~ “value2”)</td>
<td>case_match()</td>
</tr>
<tr class="even">
<td>Interactive</td>
<td>Create</td>
<td>switch</td>
<td>N</td>
<td>recode_values(x, 1 ~ column, 2 ~ column2)</td>
<td>case_match()</td>
</tr>
<tr class="odd">
<td>Interactive</td>
<td>Replace</td>
<td>if/else</td>
<td>1</td>
<td>replace_when(x, condition ~ 0, condition2 ~ NA)</td>
<td>Conditional mutate()</td>
</tr>
<tr class="even">
<td>Interactive</td>
<td>Replace</td>
<td>if/else</td>
<td>N</td>
<td>replace_when(x, condition ~ column, condition2 ~ column2)</td>
<td>Conditional mutate()</td>
</tr>
<tr class="odd">
<td>Interactive</td>
<td>Replace</td>
<td>switch</td>
<td>1</td>
<td>replace_values(x, 1 ~ 0, 2 ~ NA)</td>
<td>Conditional mutate()</td>
</tr>
<tr class="even">
<td>Interactive</td>
<td>Replace</td>
<td>switch</td>
<td>N</td>
<td>replace_values(x, 1 ~ column, 2 ~ column2)</td>
<td>Conditional mutate()</td>
</tr>
<tr class="odd">
<td>Programmatic</td>
<td>Create</td>
<td>if/else</td>
<td>1</td>
<td>vec_case_when(list(case, case2), list(“value”, “value2”))</td>
<td>numpy.select()</td>
</tr>
<tr class="even">
<td>Programmatic</td>
<td>Create</td>
<td>if/else</td>
<td>N</td>
<td>vec_case_when(list(case, case2), list(column, column2))</td>
<td>numpy.select()</td>
</tr>
<tr class="odd">
<td>Programmatic</td>
<td>Create</td>
<td>switch</td>
<td>1</td>
<td><p>recode_values(x, from = c(1, 2), to = list(“value”,
“value2”))</p>
<p>Convenience API:</p>
<p>recode_values(x, from = c(1, 2), to = c(“value”, “value2”))</p>
<p>vctrs API:</p>
<p>vec_recode_values(x, from, to)</p></td>
<td>plyr::mapvalues() + type change</td>
</tr>
<tr class="even">
<td>Programmatic</td>
<td>Create</td>
<td>switch</td>
<td>N</td>
<td><p>recode_values(x, from = c(1, 2), to = list(column, column2))</p>
<p>vctrs API:</p>
<p>vec_recode_values(x, from, to, to_as_list_of_vectors = TRUE)</p></td>
<td></td>
</tr>
<tr class="odd">
<td>Programmatic</td>
<td>Replace</td>
<td>if/else</td>
<td>1</td>
<td>vec_replace_when(list(case, case2), list(0, NA))</td>
<td></td>
</tr>
<tr class="even">
<td>Programmatic</td>
<td>Replace</td>
<td>if/else</td>
<td>N</td>
<td>vec_replace_when(list(case, case2), list(column, column2))</td>
<td></td>
</tr>
<tr class="odd">
<td>Programmatic</td>
<td>Replace</td>
<td>switch</td>
<td>1</td>
<td><p>replace_values(x, from = c(1, 2), to = list(0, NA))</p>
<p>Convenience API:</p>
<p>replace_values(x, from = c(1, 2), to = c(0, NA))</p>
<p>vctrs API:</p>
<p>vec_replace_values(x, from, to)</p></td>
<td>plyr::mapvalues(), na_if() alternative</td>
</tr>
<tr class="even">
<td>Programmatic</td>
<td>Replace</td>
<td>switch</td>
<td>N</td>
<td><p>replace_values(x, from = c(1, 2), to = list(column, column2))</p>
<p>vctrs API:</p>
<p>vec_replace_values(x, from, to, to_as_list_of_vectors =
TRUE)</p></td>
<td></td>
</tr>
</tbody>
</table>
