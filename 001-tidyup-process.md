
# Tidyup 1: the tidyup proposal process

**Champion**: Hadley Wickham  
**Status**: Proposal

## Abstract

For bigger architectural changes, particularly those that span multiple
packages or are likely to affect larger number of users, we need a
process to clearly document the problem, possible solutions, our
decisions, and to get feedback from the community. This document
describes a lightweight process for proposing changes and getting
feedback on them that I call a **tidyup**.

For now, tidyups can only be proposed by members of the tidyverse team,
but once we have more experience with the process we will to expand the
program so that any one can member of the R community can make a
proposal.

## Motivation

To date, our process for making bigger changes has been relatively
informal. This has made it easy for us to rapidly make changes but comes
with some substantial downsides:

-   There’s no standard way to invite community contribution. This means
    that we miss out on useful feedback from the community and can
    generate ill-will when changes feel like they’ve been imposed with
    no way to opt-out.

-   It’s easy to lose track of alternatives proposed and discarded early
    in the process. This can lead to substantial repeated work if we
    later discover a constraint that requires us to rethink the proposed
    solution.

-   By keeping design decisions internal, interested external developers
    can’t see “how the sausage is made” and learn from our experience.

## Solution

### Process

1.  Start by raising idea in tidyverse slack or weekly meeting.

2.  If no objections, create .Rmd (using template below) and submit PR
    to <http://github.com/tidyverse/tidyups>. Using a PR makes it easy
    for others to comment on the initial proposal.

3.  Once write up is complete, book discussion in tidyverse weekly
    meeting. Prepare a brief overview (working from the tidyup itself)
    and collect discussion items.

4.  When does PR get merged?

5.  Share with public via twitter and r-devel?. Where to discuss? GitHub
    issues? Mailing list? Level of publicity to community would depend
    on specific case.

6.  How to finish? Need to merge PR with implementation and then update
    status in tidyup repo?

### Sections

Each tidyup should have the following sections. They’re not compulsory,
but where possible it’s best to stick to the standard so it’s easier to
take in a new tidyup at a glance.

-   **Title**. Includes tidyup number and short, but evocative name.

-   **Metadata**

    -   **Champion**: each tidyup must be championed by a member of the
        tidyverse team.

    -   **Status:** draft, design approved, implemented, or declined.

-   **Abstract**. Short description of issue and proposed solution.
    Should allow the reader to determine if this is of interest to them
    and whether or not to keep reading.

-   **Motivation**. What are we doing now and why does it need to
    change?

-   **Solution.** proposed solution(s).

-   **Reference implementation**. once status is accepted, link to PR.

-   **Rationale**. Why was this solution picked? What other solutions
    were considered.

-   **Open issues**: While proposal is in process, record open issues
    that don’t yet have solutions.

-   **Backwards compatibility**. What implications does this change have
    for backward compatibility? How can any negative affects be
    ameliorated?

-   **How to teach**

### Scope

Generally, tidyups are most appropriate for big changes in interface or
implementation that might affect multiple packages. Some past cases that
would have benefited include from a tidyup are:

-   Making the pipe lazy.
-   vctrs related changes to dplyr.
-   Add case weights across tidymodels packages.

In general, a tidyup is not needed if the change only affects a single
package. There are a few exceptions:

-   If developing a new package that provides similar functionality to
    an existing package (e.g. clock, cpp11), it’s useful to have a crisp
    explanation of why we are building something new not extending an
    existing tool.

-   If the initial application is to a single package, but it might
    expand to more packages in the future, it’s worthwhile to do some
    more upfront design. Two recent examples are testthat 3e (since the
    edition idea might be used elsewhere) and `dplyr::across()` (since
    interface affects the design of functions elsewhere).

Some topics need to be written up so we understand them and can be
consistent, but don’t need to go through the full tidyup process. These
include topics like name repair, tidyverse recycling rules, ellipsis
handling (including tidy dots), and our R version policy.
