
# Tidyup 1: what is a tidyup?

**Champion**: Hadley Wickham  
**Co-champion**: ??? **Status**: Draft

## Abstract

For bigger architectural changes, particularly those that span multiple
packages or are likely to affect larger number of users, we need a
process to clearly document the problem, describe possible solutions,
record our decisions, and collect feedback from the community. This
document proposes the **tidyup** as a lightweight way of navigating that
process.

For now, tidyups can only be proposed by the tidyverse team, but we plan
to open up the process to the wider community once we have more
experience with it.

## Motivation

To date, our process for making bigger changes has been relatively
informal. This has made it easy for us to rapidly make changes but comes
with some substantial downsides:

-   There’s no standard way to invite feedback from the community. This
    means that we get useful feedback too late or not at all, and is a
    lost opportunity to build deeper engagement with the community.

-   It’s easy to lose track of solutions proposed and discarded early in
    the process. This can lead to substantial repeated work if we later
    discover a constraint that requires us to rethink the proposed
    approach.

-   By keeping design decisions internal, interested external developers
    can’t see “how the sausage is made” and learn from our experiences.

## Solution

To solve this problem, I propose a lightweight framework called a
tidyup. Tidyups follow a standard process that ultimately leads to an
.Rmd in <https://github.com/tidyverse/tidyups>. The following sections
describe the basic process, the sections that most tidyups should
contain, and a few notes about the scope of tidyups.

### Process

1.  **Propose**. Start by raising idea in tidyverse slack or weekly
    meeting. If there’s broad agreement that the scope and timing is
    right, proceed to the next step.

2.  **Write up**. Create an `.Rmd` (using sections defined below) and
    submit PR to <http://github.com/tidyverse/tidyups>. Once you’re
    happy with the write up, and your co-champion has has reviewed it,
    proceed to the next step.

3.  **Discuss**. Book a discussion in the tidyverse weekly meeting.
    Assume that everyone will at least skim the tidyup beforehand, but
    prepare to review the proposal with a focus on any parts that need
    extra discussion. Soon after the meeting, review the meeting notes
    and update the tidyup with needed changes and clarifications. If the
    proposal needs more discussion, repeat this step.

    (Depending on the complexity of implementation, the next two steps
    can be completed in either order)

4.  **Implement**. Create a reference implementation in one or more PRs
    to the appropriate repos. Update the tidyup with a link to the PRs
    in the implementation section.

5.  **Community feedback**. Ensure that you’ve rendered the `.Rmd` so
    it’s easy for others to read. Create a new issue that invites
    feedback, tag community members who’s feedback might be particularly
    helpful, set a date when the review period will end. Once the review
    period ends, update the tidyup then close the issue.

6.  **Final review**. Once both previous steps are completed, book
    another tidyverse meeting for final sign off.

7.  **Complete.** Merge implementation PRs into affected repos then
    change tidyup status to “implemented”.

### Sections

Each tidyup should have the following sections. They’re not compulsory,
but where possible it’s best to stick to the standard so it’s easier to
take in a new tidyup at a glance.

-   **Title**. Includes tidyup number and short, but evocative, name.

-   **Metadata**

    -   **Champion**: each tidyup must be championed by a member of the
        tidyverse team.

    -   **Co-champion**: every tidyup needs a co-champion who will be
        responsible for reviewing PRs.

    -   **Status:** draft, design approved (internal), design approved
        (external), implemented, declined.

-   **Abstract**. Short description of issue and proposed solution.
    Should allow the reader to determine if this is of interest to them
    and whether or not to keep reading.

-   **Motivation**. What are we doing now and why does it need to
    change? This can be long of short depending on the “obviousness” of
    the problem, how much work is needed to fix it, and whether or not
    the solution will require breaking changes.

-   **Solution(s)**. A description of the proposed solution or
    solutions. There should be enough detail to guide an implementation.
    Break this up into subsections in whatever way makes sense for the
    proposal.

    If there are multiple potential solutions to consider, each solution
    should get its own subsection that discusses the pros and cons.
    After a solution has been picked, add a conclusion that briefly
    justifies why it was picked.

-   **Open issues**: While proposal is in process, record open issues
    that require further discussion.

-   **Implementation**. Once available, provide a link to any PRs needed
    to implement the proposal.

-   **Backwards compatibility**. What implications does this change have
    for backward compatibility? How can any negative affects be
    ameliorated?

-   **How to teach.** If the change is likely to affect user code,
    include a section briefly discussing how and when to teach.

### Scope

Generally, tidyups are most appropriate for big changes in interface or
implementation that are likely to affect multiple packages. Some past
cases that would have benefited from a tidyup are:

-   Making the pipe lazy.
-   Overhauling dplyr to use strict vctrs policies.
-   Adding case weights across tidymodels packages.

In general, a tidyup is not needed if the change only affects a single
package. There are a few exceptions:

-   If developing a new package that provides similar functionality to
    an existing package (e.g. clock, cpp11), it’s useful to have a crisp
    explanation of why we are building something new and not extending
    an existing tool.

-   If the initial application is limited to a single package but it is
    likely to expand in the future, it’s worthwhile to do more upfront
    planning. Two recent examples were this would’ve applied are
    testthat 3e (since the edition idea might be used elsewhere) and
    `dplyr::across()` (since interface affects the design of functions
    elsewhere).

Some topics need to be written up so we can apply them consistently
across packages, but don’t need to go through the full tidyup process.
These include topics like name repair, tidyverse recycling rules,
ellipsis handling (including tidy dots), and our R version compatibility
policy.
