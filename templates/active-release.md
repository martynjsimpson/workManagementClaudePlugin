# Active Release

Version: TBD
Status: none

<!--
The single current delivery scope. Written by /work-plan, executed by /work-release (or
/work-spike when every item is a spike), closed out by /work-plan. Not a second backlog —
work not being built now belongs in backlog.yml.

Status values: none, proposed, approved, in-progress, testing, ready-for-release, released,
abandoned (requires an "## Abandonment note"), cancelled. Full meanings: work-model skill.

The Status line above is this release's external interface — the only signal outside the
running session of where it has got to. Both delivery commands move it as they go: a code
release runs approved -> in-progress -> testing -> ready-for-release, a spike release runs
approved -> in-progress -> ready-for-release. Work item statuses below it are not a
substitute; closeout reads the top-level line alone.

Version and branch handling — assignment timing, the version-before-branch-name ordering,
and the Branch: line recorded beside Version: above — are specified in /work-release Steps
2 through 2b. This file only ever records the outcome; it never decides it.

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
  Boundaries within this release's own theme — what a reasonable reader might
  assume is included but is not. Not a list of backlog items that were considered
  and not selected; that is backlog.yml's job. 0-3 bullets, or "None."

  ## Required agents
  Names from project.yml agents[], each with what it is needed for.

  ## Verification bar
  Per item, what must be true to call it done.

  ## Blockers

  ## Deferred items for PM
  Added during a release when something out of scope surfaces. /work-plan triages each
  entry — a decision becomes a request, specified work goes straight to backlog.yml with
  source_release set, and a note-only finding goes to the durable record — then removes
  the block. Write each entry with enough detail to act on without this session.
-->

No active release.
