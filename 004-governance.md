
# Tidyup 4: governance model

**Champion**: Hadley Wickham  
**Status**: Proposal

## Abstract

## Motivation

We currently have one package (ggplot2) with a [formal governance
model](https://github.com/tidyverse/ggplot2/blob/master/GOVERNANCE.md),
and an informal, undocumented model that we’ve used for other packages
(e.g. dtplyr, dbplyr). Goal is to come up with holistic, flexible model,
that can become our default governance model for all packages going
forward. This is part of a general movement to get better at defining
and documenting our processes to so that there’s a clear path (if steep
at times!) from package user to package developer.

## Solution

This document describes a default governance model to use for open
source RStudio repositories (starting with those in the r-lib,
tidyverse, and tidymodels organisations). It is not mandatory, but it
has been designed to reflect our current best practices, and should be
used unless there are compelling reasons to favour a different approach.

Our model is inspired by (and heavily adapted from) the [benevolent
dictator governance
model](http://oss-watch.ac.uk/resources/benevolentdictatorgovernancemodel)
by Ross Gardler and Gabriel Hanganu. We have built on this model because
we believe that packages are best shaped by a single voice, and few
packages are of sufficient scope to benefit from team management.

For maintainer of many of our open source repositories are RStudio
employees. We want to acknowledge the a tension between making
development open to all and ensuring that users can trust that a package
will be maintained in the long term (i.e. 10+ years), which typically
requires the maintainer be explicitly remunerated for their work. Where
the maintainer is not an employee of RStudio, we ask for the “right of
first refusal” — if the maintainer wants to stop maintaining the package
(for whatever reason) they first offer it back to RStudio. In the
future, we hope to find other ways of financial supporting maintainers
apart from full-time employment.

Our model includes four key roles that are described in details below: a
large community of package **users,** a smaller pool of GitHub
**contributors**, a team of **authors**, and a single **maintainer**.
All roles are bound by the code of conduct.

### Users

People who use the package are the most important members of the
community; without these users, this project would have no purpose.
Users are encouraged to participate in the life of the project and the
community as much as possible as user contributions help ensure that the
project is useful.

Common user activities include (but are not limited to):

-   Evangelising about the project.
-   Asking and answering questions on community forums.
-   Providing moral support (a “thank you” goes a long way).

### Contributors

Users who continue to engage with the project and its community will
often find themselves becoming more and more involved. Such users may
then go on to become **contributors** by interacting with the project on
GitHub. Contributors:

-   Report bugs and suggest improvements by creating new issues.

-   Improving existing issues by answering questions, creating reprexes,
    or providing feedback on proposed changes.

-   Contributing code or documentation via pull requests.

Anyone can become a contributor: there is no expectation of commitment
to the project, no required set of skills, and no selection process. We
do not maintain an explicit list of contributors but acknowledge them in
blog posts using `usethis::use_tidy_thanks()`, which gathers
contribution data from the GitHub API.

### Authors

Contributors who have made significant and sustained contributions can
be invited to become authors. Authors are collectively responsible for
day-to-day development of the package, including responding to issues
and reviewing pull requests. An author possesses two special powers:

-   They have **write** access on GitHub so they can triage issues,
    request review on PRs, and merge them.

-   They are listed in `Authors@R` so they receive credit when others
    cite the package.

Authors are expected to follow our standard processes, such as:

-   **Code contribution**: code is usually contributed via PR, even for
    authors who could push directly. Particularly high-stakes project
    may want to protect the main branch and require “request reviews
    before merging”.

-   **Communication**: authors are involved in most of the interactions
    with contributors and thus need to set a welcoming and inclusive
    tone for the project.

-   **PR review**: all pull requests should be reviewed by at least one
    other author. PRs are usually squashed-merged so that individual
    contributors don’t need to worry about maintaining a clean history.

-   **Backward compatibility**: any backward incompatible changes
    (i.e. changes that cause reverse dependencies to fail `R CMD check`
    or are likely to cause problems in user code) must be approved by
    the maintainer. Significant backward incompatible changes need to be
    accompanied with a plan for how they will be communicated to the
    community.

-   **CRAN releases**: package releases are made on an as-needed basis,
    and increment either the major, minor, or patch version depending on
    the scope of the release. The process itself is defined by
    `usethis::use_release_issue()`.

-   **Decision making:** decisions are made using a consensus model
    where authors and contributors consider and discuss decisions in
    GitHub issues. The maintainer reserves the right to make a final
    decision in contentious cases. If the community questions a
    decision, the maintainer may review it and either uphold or reverse
    it.

(We expect to flesh these processes out in the coming months.)

Authors are recruited from contributors. An invitation to join the
authors can be extended to anyone who has made significant and sustained
contributions, and has acted in accordance with the code of conduct. Any
existing author can propose a contributor be invited team by emailing
the maintainer.

### Maintainer

A maintainer is the author with primary responsibility for the project.
As well as the responsibilities of an author, they also:

-   Set and clearly communicates the strategic objectives of the
    project.
-   Oversee CRAN releases.
-   On-board new authors.
-   Mediate conflicts amongst authors.
-   Enforce the code of conduct.
-   Recruit a new maintainer when ready to retire from the project.

The maintainer is listed in `Authors@R`. They must list their email
address and be identified by the “cre” (creator) role. They have
**admin** access on Github, allowing them to add new authors when
needed.

Maintainer turnover is slow and we have not yet developed a process for
it, but we’d generally expect a maintainer to be a long-standing author.

### Common tasks

#### Author invitation

To on-board a new author, the maintainer first emails all other authors,
checking for any major concerns. If all authors are agreeable (or don’t
response within 7 days), the maintainer then sends the following email:

> Hi {name},
>
> In recognition of your significant contributions to {package}, would
> you be interested in becoming a co-author of the package? This means
> that you’ll be acknowledged in <Authors@R> and given write permission
> on GitHub. Write permission gives you the power to change dbplyr
> directly (which is fine for smaller fixes) but I’d appreciate it if
> you’d continue to send major stuff through the pull request process.
>
> If you accept, can you please prepare a PR that:
>
> -   Adds your info to `Authors@R`
> -   Tweaks `_pkgdown.yml` if you want to link somewhere from the
>     pkgdown site
> -   Advertises the change in NEWS.md
> -   Re-builds the documentation to get updated package docs
>
> I’ll then add you as admin, and approve the PR, then you can
> squash-merge it. This would be our workflow going forward. (You’ll
> also be able to request reviews from me and other authors as needed).
>
> Thanks for all your work on {package}!
>
> {your\_name}

(After this tidyup is approved, the template will become a usethis
function.)
