
# Tidyup 5: Github simplification

Champion: Hadley Wickham  
Co-champion: ???  
Status: draft

## Abstract

## Motivation

Originally, we divided tidyverse and r-lib because we thought it was
important to keep data science and package development tools separate.
Then when tidymodels kicked off, it seem natural for it to have its own
home (and then other teams at RStudio copied this principle). But there
are some downsides to having multiple organisations for the tidyverse
team:

-   While the distinction between tidyverse, r-lib, and tidymodels is at
    reasonably strong in our heads, it’s hard for users to articulate
    these differences, and to learn/remember exactly which organisation
    a package belongs to. This is particularly hard when you look
    holistically at all packages made by RStudio.

-   There are a few packages that I am confused about:

    -   Is rvest programming or data science? what about httr?
    -   usethis is technically programming, it’s very user facing, and
        we expect data scientists might use it for analysis projects.
    -   glue is useful for data science but designed to be zero dep
        specifically for use by package developers.
    -   Both tibble and magrritr are technically user-facing, but most
        people use via other packages. tibble is now really mostly about
        defining a data structure.
    -   modelr is package that initially made sense in tidyverse, but is
        now mostly deprecated.
    -   googledrive is not really about data analysis or package
        development.
    -   vroom is in r-lib, readr is in tidyverse.

-   As we start to use GitHub teams more, managing separate teams across
    separate orgs will become more painful. It also makes it hard to
    centralise discussions, and to succinctly explain the scope of
    packages that tidyups apply to.

-   It’s painful to move issues across orgs. This affects packages like
    dplyr where the underlying issue might need to be fixed in vctrs.

## Solutions

I think there are four possibilities:

-   Move r-lib and tidymodels repos to **tidyverse org**.
-   Move tidyverse and tidymodels repos to **r-lib org**.
-   Move r-lib, tidyverse, and tidymodel repos to a **new org**.
-   Move r-lib, tidyverse, and tidymodel to the **rstudio org**.

These cases only affect GitHub organisations and in the short term all
other websites would stay as is. This means that (e.g.) vroom would
continue to live at vroom.r-lib.org. In the longer term, we’d probably
make it vroom.{orgname}.org, but this would required some thought about
how to manage the redirects, and whether it makes sense for (e.g.)
parsnip to live at parsnip.{orgname}.org while the main tidymodels
website continues to live at tidymodels.org.

### Move to tidyverse org

Pros:

-   Moves smallest number of packages.
-   Tidyverse has strongest brand identity.

Cons:

-   Worsens the existing semantic overloading of tidyverse: it’s a
    package, it’s a set of packages loaded by `library(tidyverse)`, it’s
    a set of packages installed by `install.packages("tidyverse")`,
    **and** it’s a GitHub organisation.

    Could work to counter this with consistent tags, website themes, and
    more details in README. It would be useful to include a block at the
    top of the readme aimed at the non-expert Github users that was not
    shown on the package website.

-   Binds all packages more closely to tidyverse. This might be less
    appealing for external developers (e.g. Jeroen Ooms, Michel Lang)
    and for other RStudio teams we might hope would also migrate to one
    org.

    Particularly problematic for gert, which is an ROpenSci project, so
    if we move to tidyverse, it would probably need to move to a
    different repo.

-   No longer a one-to-one relationship between package homepage and
    GitHub organisation. (But could resolve in a second phase).

-   Lose ability to bury less-important packages in r-lib; i.e. harder
    to highlight most important packages in tidyverse.

### Move to r-lib org

Pros:

-   Existing org name is more generic

Cons:

-   Little awareness of r-lib “brand”

-   Might be perceived as muscling in on R itself.

### Create a new org

Pros:

-   May be easier to persuade other organisations to join if not
    “tidyverse” branded.

Cons:

-   Maximum amount of work moving packages and renaming url.
-   Will be hard to come up with good name.
-   Build up brand from scratch.
-   In the short term, increases numbers of orgs.

### Move to rstudio org

Pro:

-   Greatest reduction in organizations.

-   Easy story to tell (now that RStudio is a PBC, we have 0% concerns
    about it as long term steward of this code).

Cons:

-   Eliminates veneer of separation between RStudio and tidyverse, and
    makes it harder to invite/recruit non-employee authors and
    maintainers.
-   Would fall under RStudio IT practices which are geared towards
    protecting private data. This will make it challenging to manage our
    own processes around teams.
-   Would need very clear reassurance that this isn’t RStudio “muscling
    in” on open source. Regardless of how we communicate this, a few
    people are likely to generate negative press.

## Open questions

-   GitHub’s existing forwarding should handle most cases, but there a
    few cases where you’ll need to update the urls in the GitHub repo.
    But are there any existing workflows that this would break?
-   Should we consider the moving the websites as part of this process?
    Yes. If so, how do we manage the redirects? Will need to set up some
    external server.
-   Should r-dbi be included? Probably not because it includes DBI which
    is very much not RStudio or tidyverse.
-   How do we clearly communicate what’s happening, and how do we update
    how we have described “tidyverse” in various places?
-   If we choose option 1, how does this affect future authors of a
    putative 2nd edition of tidyverse paper?