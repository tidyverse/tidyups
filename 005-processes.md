
# Tidyup 5: standard processes

**Champion**: Hadley Wickham **Status**: Proposal

## Abstract

## Motivation

## Solution

These are defaults; they’re not required, but since they’re the
convention, if you depart from them you’ll need to document how. This
documentation should go in `CONTRIBUTING.md`.

Smaller conventions on the more technical side can be found in
<https://github.com/r-lib/usethis/issues/1416#issuecomment-821308447>.
Many of the smaller components are not explicitly mentioned in this doc.

### Authorship

See 004-governance for details. In brief we use the following terms:

-   **Contributors** are thanked in blog posts, but not recorded in the
    package.

-   **Authors** are listed in `Authors@R`.

-   The (one) **maintainer** is listed listed in `Authors@R` with role
    “cre”,
    e.g. `person(given = "Hadley", family = "Wickham", role = c("aut", "cre"), email = "hadley@rstudio.com")`.

-   Additionally, for packages written primarily by RStudio employees,
    we list RStudio as copyright holder and funder:
    `person(given = "RStudio", role = c("cph", "fnd"))`.

### Code contribution

Code is usually contributed via PR, even for authors who could push
directly. Particularly high-stakes project may want to protect the main
branch and require “request reviews before merging”.

All pull requests should be reviewed by at least one other author. Helps
both people learn, and ensures that every project has at least one
person apart from the maintainer with some idea of what’s going on. The
primary exception to this rule is when a package is still very young,
and the interface and internals are changing rapidly. Even in this
situation it’s a good idea to still make a PR, let it rest overnight,
and then re-read the next morning.

PRs are usually squashed-merged so that individual contributors don’t
need to worry about maintaining a clean history. Rewrite commit message
if that happens:
<https://style.tidyverse.org/gitgithub.html#commit-messages>. If a PR is
larger and the author has carefully created a clean history, you can

### Licensing

Where possible, we use the MIT license. This is a simple, well
understood license that makes our open source code as easy for others to
use a possible. Because it’s very permissive, there’s no need for a CLA.

There are some exceptions: if bundling open source code with other
licenses will need to ensure that the package license is compatible.
This most commonly arises with GPL C code; in that case, just match the
license of the package to the license of the bundled code.

Whenever you bundled any code, the primary authors must be listed in
`Authors@R`, and the licenses listed in `LICENSE.note`. See details in
<https://r-pkgs.org/license.html#code-you-bundle> for more details and
best practices.

The easiest way to add an MIT license is to use
`usethis::use_mit_license()`. This was revamped at {DATE}, so if you
have an older package check that the `LICENSE` uses the updated author
wording (“all authors of the package”). There is no need to regularly
update the copyright year, but you will need to check it prior to
initial CRAN submission.

### Backward compatibility

Any backward incompatible changes (i.e. changes that cause reverse
dependencies to fail `R CMD check` or are likely to cause problems in
user code) must be approved by the maintainer. Significant backward
incompatible changes need to be accompanied with a plan for how they
will be communicated to the community.

### Release process

Package releases are made on an as-needed basis, and increment either
the major, minor, or patch version depending on the scope of the
release. The process itself is defined by
`usethis::use_release_issue()`.

### Style

Follow the advice in <http://style.tidyverse.org/>

### Documentation

It’s important! Put time into it. We use roxygen2. Conventions at
<https://style.tidyverse.org/documentation.html>.

Every site should have a pkgdown website build automatically by GitHub
actions and hosted on GitHub pages. Url should be either
`{pkgname}.{site}.org` or `pkgs.rstudio.com/{pkgname}`. Listed in URL
field of DESCRIPTION.

Readme should follow template in `usethis::use_readme_rmd()`. Start with
a brief overview of the package, then get into a meaningful example as
quickly as possible.

Readme should include code of conduct at bottom.

News file following <https://style.tidyverse.org/news.html>.

### Testing

We use testthat 3e, along with covr for test coverage. Don’t strive for
100% test coverage, but use it tactically to double check that your
tests cover what you think they cover. Test style in
<https://style.tidyverse.org/tests.html>.

### Issues

Batched processing. Regular triage.

Welcoming and inclusive, but efficient. Invest time into carefully
crafted responses that we re-use a lot.