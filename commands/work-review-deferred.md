---
description: Re-triage every deferred request and work item — keep parked, reclassify as blocked, promote, or reject.
---

Act as the Product Manager for one narrow purpose: review everything currently deferred and
decide what it should actually be. This is not a planning session. Do not propose a
release, refine requests, or touch the active release.

Deferred items are deliberately invisible to `/work-plan` — that is what distinguishes them
from blocked items. This command is the only thing that surfaces them, so it is the only
place a wrongly-parked item gets caught.

## Step 0 — Resolve configuration

Follow `${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/config-resolution.md`. Read the
manifest at `<paths.work>/project.yml`. Refuse to proceed if it is missing.

## Step 1 — Collect

Read `<paths.work>/requests.md` and `<paths.work>/backlog.yml`. Collect every request with
`Status: deferred` and every work item with `status: deferred`.

If there are none, say so and stop.

## Step 2 — Walk through them

Present the items as a readable list — ID, title, and any existing note — grouped by theme
where that helps. For a long list, work through it in batches rather than dumping all of it
at once.

For each item, offer these four outcomes:

**Keep deferred** — genuinely parked, no dependency, not under consideration. Leave it
untouched.

**Reclassify as blocked** — something specific has to happen first. Ask what, and record
the answer verbatim in `Blocked on:` / `blocked_on`. A blocker recorded as "waiting on
other work" is useless; it has to name the thing. Set the status to `blocked`.

**Promote** — worth doing now or soon. Move a request to `## Inbox / needs refinement` with
`Status: needs-refinement`; set a work item to `needs-refinement`. `/work-plan` picks it up
next session.

**Reject** — no longer relevant or explicitly out of scope. Move a request to the
deferred/rejected section with `Status: rejected` and a one-line reason. For a work item,
either set `status: deferred` with a rejection note, or remove it entirely if it has no
history worth keeping.

Where an item's original rationale is no longer legible, say so rather than guessing — the
right answer is often to reject it and re-capture the underlying need fresh.

## Step 3 — Write the changes

Once every item is classified, write all changes in one pass. Move each request to its
correct H2 section, update statuses, and add or remove `Blocked on:` / `blocked_on` as
appropriate. Never leave a `blocked` item without a named blocker, and never leave
`blocked_on` on a `deferred` item.

Report the totals: how many stayed deferred, became blocked, were promoted, were rejected.

## Constraints

- Do not touch `<paths.work>/active-release.md`.
- Do not propose a release or select work items.
- Do not refine anything beyond reclassifying its status — promoting an item hands it to
  `/work-plan`, it does not mean writing its acceptance criteria now.
- Write only to `<paths.work>/requests.md` and `<paths.work>/backlog.yml`.
- Do not edit `<paths.work>/project.yml`.
