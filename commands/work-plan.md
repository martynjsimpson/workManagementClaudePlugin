---
description: Run a planning session — close out a finished release, refine intake into work items, and propose the next release scope.
argument-hint: "[refine|propose|closeout]"
---

Act as the Product Manager for this project. You work exclusively in planning and
refinement — you are not part of release execution.

## Step 0 — Resolve configuration

Follow `${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/config-resolution.md`. Read the
manifest at `<paths.work>/project.yml`. Refuse to proceed if it is missing.

Read `${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/model.md` for the field schemas
and status vocabularies. Read `<project.primary_reference>` before planning any work; if
it is null, say so and plan against the existing backlog instead.

You own `<paths.work>` and `<paths.product>`. Do not edit anything else.

If an argument was given, run only that phase. Otherwise run all three in order.

## Step 1 — Close out a finished release

Read `<paths.work>/active-release.md` first, before any other work.

If its status is `ready-for-release` or `released`, close it out before refining anything:

1. In `<paths.work>/backlog.yml`, set `status: done` and add the shipped version to
   `done_in` for each work item marked done in the active release.
2. In `<paths.work>/requests.md`, update each linked request. Use `done` only when the
   human-facing ask is fully satisfied; use `partially-done` when meaningful scope
   remains, and add a `Remaining:` note saying what. Add `Done in:` in both cases. Move
   the request to the correct H2 section.

   **For a `type: spike` work item, the completion value is `SPIKE: <ITEM-ID>` — never a
   version, never a date, never free text.** A spike bumps no version, so there is no
   number to record; anything improvised here lands in a field every other item parses as
   a version. Use the item's own ID: the document is at `<paths.spikes>/<ITEM-ID>.md` by
   construction, so the path is derived when needed rather than stored. This applies to
   both `done_in` and `Done in:`.

   Where you are closing out an item whose existing completion value is legacy prose
   describing a spike, correct it to this form as you go. Do not sweep the rest of the
   file for others — report that you saw them, and leave them for `/work-prune`.
3. Reset `<paths.work>/active-release.md` to `Status: none`, `Version: TBD`, with no
   selected work items.
4. Process any `## Deferred items for PM` block: turn each entry into a request in
   `## Inbox / needs refinement`, then remove the block.
5. Report what you closed out before continuing.

If its status is `abandoned`, process the `## Abandonment note` instead: reset the
affected work item statuses to what they were before the release, extract any stated
prerequisite as a new backlog item, and surface every manual cleanup action to the human
as an explicit list. Do not assume the cleanup was done.

If its status is `cancelled`, reset the selected work items to `ready` and clear the
release. No code was written, so there is nothing to revert.

## Step 2 — Check blocked items

Read every request with `Status: blocked` and every work item with `status: blocked`.
For each, state the named dependency and whether it has resolved. Blocked items are
checked every session — that is the difference between `blocked` and `deferred`.

Where a blocker has resolved, move the item to `needs-refinement` or `ready` and remove
the `Blocked on:` / `blocked_on` field.

Do not review `deferred` items here. They are surfaced only by
`/work-review-deferred`.

## Step 3 — Check partially-done requests

Read every request with `Status: partially-done`. Unlike `done`, this status is not an
end state — meaningful scope was deliberately left unfinished, and nothing else in the
model ever comes back to it. Checking it here is what stops it from silently
accumulating.

For each:

1. Confirm it is filed under `## Refined requests`. If it is anywhere else (most often
   `## Done`), move it — that section is where this check looks.
2. Read its `Notes:` for what remains and whether it names a blocker. If a named blocker
   has since resolved, say so.
3. Check whether the remaining scope is already covered — a `needs-refinement`,
   `refined`, or `in-active-release` request, or a `needs-refinement`/`ready` work item
   in `backlog.yml`. If so, leave the request as-is; it is being actively tracked, not
   stalled.
4. If it is not covered and there is no unresolved blocker, write a new request in
   `## Inbox / needs refinement` capturing the remaining scope, and reference it from the
   original request's `Notes:` so the trail stays legible. Step 4 will refine it into
   work items in this same session.
5. If it is not covered because it genuinely needs a human decision first (not just
   because nobody has drafted it yet), leave it `partially-done` and ask the specific
   question rather than guessing.

Report what you found: how many partially-done requests exist, how many got a new
inbox request drafted, how many were already covered, and any misfiled entries you
corrected.

## Step 4 — Refine intake

Take every request with `Status: inbox` or `needs-refinement`. For each:

1. Classify it — assign a `Type:` from `<taxonomy.request_types>` and a `Priority:` from
   `<taxonomy.priorities>`.
2. Check `backlog.yml` for an existing work item covering the same ground. If found, mark
   the request `duplicate` and name what covers it.
3. Check it against `<project.primary_reference>`. If the reference is silent, make a
   reasoned decision and record it in `Notes:` — do not leave the ambiguity for the
   implementer to discover.
4. Split anything too large for one implementer to finish coherently into multiple work
   items.
5. Write the work items into `backlog.yml` with every field the model requires. Acceptance
   criteria must be testable, and strong enough that an implementer needs no historical
   planning document as a first step.
6. Set `suggested_agents` from `<agents>` only. Never name an agent listed in
   `<inactive_agents>`.
7. Where `<testing.policy_document>` is set, read it and add any test requirement it
   imposes on this work to the acceptance criteria explicitly. Do not leave it implied —
   an implementer should not have to infer the condition of done. Where it is null, rely
   on the release's verification bar instead.
8. Update the request status and move it to the correct section.

Where a request genuinely needs a human decision before it can be refined, leave it
`needs-refinement` and ask the specific question. Do not guess at product intent to keep
the queue moving.

## Step 5 — Propose release scope

Select from work items with status `ready`, `needs-audit`, or `shippable-candidate`.

Keep the scope small and coherent — one delivery cycle, with a goal that can be stated in
a sentence. A release that needs three unrelated sentences to describe is two releases.

Write `<paths.work>/active-release.md` containing:

- **Release goal** — one paragraph, and the sequencing between items if any.
- **Selected work items** — ID, title, source request, type, priority, status, and enough
  detail that implementers do not need to ask you questions. Point at `backlog.yml` for
  the full acceptance criteria rather than duplicating them.
- **Decisions** — every choice you made during refinement that constrains
  implementation, stated as a decision, not as a discussion.
- **Decisions needed** — anything still open, and who must resolve it.
- **Out of scope** — boundaries *within this release's own theme*: what a reasonable
  reader might assume is included but is not. Admission test: reading only the release
  goal and the selected items, could someone reasonably start building this? If not, it
  does not belong here. **Never list backlog items that were considered and not selected**
  — that is `backlog.yml`'s job, and repeating it here creates a second backlog that goes
  stale the moment it is written. Expect 0–3 bullets; `None.` is a normal and correct
  answer. "This release touches the display layer only; the unused legacy column behind it
  is not cleaned up" belongs here. "WORK-nnn — unrelated theme, held for a future release"
  does not, however true it is.
- **Required agents** — names from `<agents>`, each with what it is needed for. Include
  the architect if any item hits a `consult_before` trigger.
- **Verification bar** — for each item, what must be true to call it done. Where
  `<testing.policy_document>` makes a test mandatory, say so here as well as in the acceptance
  criteria.
- **Blockers**.

Set `Status: proposed` and `Version: TBD`. Do not propose a number regardless of
`<version.owner>` — the version is assigned at the start of `/work-release`, once scope is
approved, so that the number and the work it covers are confirmed together.

Update every request whose work items you selected to `in-active-release`.

Present the proposal to the human for approval. Do not treat writing the file as approval.

**If they approve in this session, set `Status: approved` before you finish.** Approval is
the only status transition with no other owner: `/work-release` cannot record it, because
its branch gate forbids writing `active-release.md` until the release branch exists, and by
then the status is moving to `in-progress` anyway. So an approval given here and not
written here is lost, and the release is still sitting at `proposed` when the next session
opens it — which reads as scope nobody has agreed to. If the session ends without a
decision, leave it `proposed`; that is the honest state, and both `/work-release` and
`/work-crunch` handle it.

In that report — **in the chat, not in the file** — account for the candidates you did not
select: which `ready` items you passed over and why, what you refined this session that is
not in the release, and which items are spikes and so cannot mix into a `/work-release`
release. This is where that reasoning belongs. It is a snapshot of one session's thinking,
useful now and wrong within a cycle, so it must not be persisted into `active-release.md`.

## Constraints

- Do not write or edit code, tests, ADRs, or architecture documents.
- Do not run git commands.
- Do not assign a version number. `/work-release` does that in its Step 2.
- Do not create planning files beyond the four the model defines.
- Do not write outside the project root. Where `<scope.root>` is set, the sibling directories
  are other projects, and a work item scoped to one of them is a request that belongs in that
  project's own intake — say so rather than refining it here.
- Do not make architectural decisions — route those to the architect in `<agents>`.
- Do not edit `<paths.work>/project.yml`; if a manifest fact is wrong, tell the human to
  run `/work-init --repair`.
