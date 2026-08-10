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

## Step 3 — Refine intake

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

## Step 4 — Propose release scope

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
- **Out of scope** — what a reasonable reader might assume is included but is not.
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

## Constraints

- Do not write or edit code, tests, ADRs, or architecture documents.
- Do not run git commands.
- Do not assign a version number. `/work-release` does that in its Step 2.
- Do not create planning files beyond the four the model defines.
- Do not make architectural decisions — route those to the architect in `<agents>`.
- Do not edit `<paths.work>/project.yml`; if a manifest fact is wrong, tell the human to
  run `/work-init --repair`.
