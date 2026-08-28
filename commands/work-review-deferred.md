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

Present the items as a readable list — ID, title, any existing note, and **how long it has
been parked and how many times it has been reviewed**, from `Parked since:` / `Reviewed:`
(`parked_since` / `reviewed` on a work item). Group by theme where that helps. For a long
list, work through it in batches rather than dumping all of it at once.

The count is not decoration. An item reviewed repeatedly without anyone acting on it is
evidence about how much it is wanted, and surfacing that evidence is most of what this
command is for — see "Review pressure" in `model.md`.

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

**At `Reviewed:` 5 or more, "keep deferred" stops being a neutral option.** Say the count
out loud, and put the choice as reject, promote, or park against a **named trigger** — a
specific thing that would make the item live again. Where the trigger is a dependency the
item becomes `blocked` with that dependency in `Blocked on:` / `blocked_on`; where it is a
condition, record it in `Notes:` and reset `Reviewed:` to 0, because the item is now
waiting on something stated rather than on nobody having decided. Keeping it deferred
unchanged a sixth time is the one answer to argue against, and say why: five reviews is a
long time for something nobody has wanted enough to schedule.

Where an item's original rationale is no longer legible, say so rather than guessing — the
right answer is often to reject it and re-capture the underlying need fresh.

Where the rationale is legible but unreadable — a title or summary that only makes sense
with the code open — rewrite it to the request-voice rules in `model.md` before asking for
a decision. Do not ask the human to rule on something they cannot parse.

## Step 3 — Write the changes

Once every item is classified, write all changes in one pass. Move each request to its
correct H2 section, update statuses, and add or remove `Blocked on:` / `blocked_on` as
appropriate. Never leave a `blocked` item without a named blocker, and never leave
`blocked_on` on a `deferred` item.

For every item that stayed deferred, set `Parked since:` if it has none, then increment
`Reviewed:` and **replace** the line rather than appending a fresh paragraph to `Notes:`
restating the item. Append to `Notes:` only when something materially changed since the
last review. Clear both fields on anything promoted or rejected.

Report the totals: how many stayed deferred, became blocked, were promoted, were rejected —
and, separately, how many are now at `Reviewed:` 5 or more, since that is the set worth
looking at first next time.

## Constraints

- Do not touch `<paths.work>/active-release.md`.
- Do not propose a release or select work items.
- Do not refine anything beyond reclassifying its status — promoting an item hands it to
  `/work-plan`, it does not mean writing its acceptance criteria now. Rewriting an
  unreadable `Title:` or `Summary:` into the request's own voice is not refinement and is
  allowed; it changes how the ask is stated, not what it is.
- Write only to `<paths.work>/requests.md` and `<paths.work>/backlog.yml`.
- Do not edit `<paths.work>/project.yml`.
