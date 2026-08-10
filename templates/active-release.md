# Active Release

Version: TBD
Status: none

<!--
The single current delivery scope. Written by /work-plan, executed by /work-release
(or /work-spike when every item is a spike), closed out by /work-plan.

This is not a second backlog. If it starts listing work that is not being built now,
that work belongs in backlog.yml.

Status values:
  none                no release in flight
  proposed            scope drafted, awaiting human approval
  approved            scope approved, implementation not started
  in-progress         implementation under way
  testing             implementation complete, verification under way
  ready-for-release   verified, awaiting the release action
  released            shipped, version recorded
  abandoned           started but not completed, AND all code reverted
                      -> requires an "## Abandonment note"
  cancelled           scope withdrawn before implementation started; no code written

Version assignment follows version.owner in project.yml. /work-plan always writes
Version: TBD. /work-release assigns the real number in its Step 2 — immediately after
scope approval, reporting it without writing anything — and writes it to version.file
and version.mirrors in Step 2b, after the release branch exists and before any
implementation work begins.

The order matters where vcs.branching is `branch` or `pr`: nothing is written until the
branch is confirmed, because a write before that lands on whatever branch the repository
is sitting on. The branch name cannot be chosen before the version, since it contains it
(release/<project-slug>-<version>) — so version first, then branch, then writes.

Where vcs.branching is not `none`, /work-release records the branch it is building on as
a Branch: line beside Version: above, so the release is traceable to it after the fact.
What that branch was CUT FROM is not recorded and not asked about — the base is the
human's call, and on a shared repository the default branch is often the wrong answer.
Add it by hand if a particular release needs it stated.

Section skeleton for a proposed release:

  ## Selected work items
  ### <WORK-ID> — <title>
  Source: <REQ-ID>
  Type: <type> | Priority: <priority> | Status: <status>
  <what is being built, and the constraints on it. Point at backlog.yml for the
  full acceptance criteria rather than duplicating them.>

  ## Decisions
  Choices already made that constrain implementation. Stated as decisions, not
  discussions.

  ## Decisions needed
  What is still open, and who resolves it.

  ## Out of scope
  What a reasonable reader might assume is included but is not.

  ## Required agents
  Names from project.yml agents[], each with what it is needed for.

  ## Verification bar
  Per item, what must be true to call it done.

  ## Blockers

  ## Deferred items for PM
  Added during a release when something out of scope surfaces. /work-plan converts
  each entry into a request and removes the block.
-->

No active release.
