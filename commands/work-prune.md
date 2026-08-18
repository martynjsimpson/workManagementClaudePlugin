---
description: Prune completed requests and work items whose history is already durably recorded, keeping the work files focused on live work.
argument-hint: "[before <version>]"
---

Trim completed items out of the work files. The model is an operating model for live work,
not an archive — the durable record of shipped work is the changelog, git history, release
tags, tests, code, and decision records.

## Step 0 — Resolve configuration

Follow `${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/config-resolution.md`. Read the
manifest at `<paths.work>/project.yml`. Refuse to proceed if it is missing.

## Step 1 — Establish the cutoff

If an argument gave a version, use it. Otherwise, read the recent version history from
`<paths.changelog>` (or from release tags if `<paths.changelog>` is null and `<vcs.system>`
is `git`), propose a cutoff that retains roughly the last three releases, and confirm it.

**Filter the tag history by `<version.tag_template>`** where it is set. A repository's tag
list holds every member's releases, and reading it unfiltered gives a version history
belonging to several projects at once — which produces a cutoff that is either far too recent
or far too old, and in both directions prunes the wrong things. Match the template's literal
parts and read the version out of the `{version}` position; where the template is null, the
tags are bare versions and the list is this project's alone.

If no durable record exists, stop — there is nothing to prune *into*, and removing the items
would destroy the only copy.

**When `<vcs.system>` is `none`, the changelog is the entire durable record.** There is no
commit history and no tag history to fall back on, so:

- refuse outright if `<paths.changelog>` is null — pruning would be pure data loss;
- do not offer tags as an alternative source for the version history;
- retain more rather than less. Propose a cutoff that keeps the last five releases rather
  than three, and say why;
- treat the Step 3 changelog verification below as an absolute gate, not a warning.

## Step 2 — Identify candidates

A request may be pruned only when all of these hold:

- `Status: done`;
- every version in `Done in:` is older than the cutoff;
- it has no unresolved remaining work.

A work item may be pruned only when all of these hold:

- `status: done`;
- every version in `done_in` is older than the cutoff;
- no live work item lists it in `dependencies`.

**Never prune** anything with status `partially-done`, `deferred`, `blocked`,
`needs-audit`, `needs-refinement`, `ready`, `in-progress`, `needs-test`, or
`in-active-release`.

**A `SPIKE: <ITEM-ID>` completion value is not a version.** Do not compare it to the cutoff
and do not treat it as failing the cutoff test — a spike bumps no version, so it has no age
to measure. An item carrying one is a candidate on the strength of the Step 3 document check
below, subject to the same dependency and referential rules as anything else. An item
carrying both a spike marker and one or more versions is age-gated on the versions alone.

Where a completion value is neither a version nor a `SPIKE:` marker — legacy free text like
`spike completed 2026-08-03 (no release/code change)` — do not prune the item and do not
attempt to interpret the prose. List it for the human to correct, naming the form it should
take.

## Step 3 — Verify the durable record before removing anything

For each candidate, confirm its outcome is actually represented in `<paths.changelog>` (or,
under `git`, in the tag history filtered per Step 1). This check is the point of the command —
pruning an item whose delivery was never recorded loses the information permanently, and a
`done` status is not by itself evidence that the changelog was updated.

Where a candidate has no changelog representation, do not prune it. List it separately as a
changelog gap for the human to fix first.

**For a candidate carrying `SPIKE: <ITEM-ID>`, the durable record is the spike document, not
the changelog.** Check that `<paths.spikes>/<ITEM-ID>.md` exists and has both a `## Findings`
and a `## Recommendations` section populated; that is the same verification, pointed at the
right record. If it is missing or thin, do not prune — the investigation's output is the only
thing the item produced.

Never report a spike-completed item as a changelog gap. A spike ships no code and bumps no
version, so it correctly has no changelog entry, and asking the human to write one sends them
to add a record that should not exist.

Also check referential integrity: if a pruned work item is referenced by a retained
request's `Work items:` line, or vice versa, either prune both or neither. Do not leave
dangling IDs.

## Step 4 — Report, confirm, then remove

Present, before changing anything:

- the cutoff;
- requests to be pruned (ID, title, `Done in:`);
- work items to be pruned (ID, title, `done_in`);
- items deliberately retained despite being done, and why;
- changelog gaps blocking a prune.

Get explicit approval. Then remove the approved items in one pass, leaving the section
headings and the legend intact. If a section empties, leave the heading with `*(empty)*`
beneath it — the four sections are part of the file's required structure.

Report the line counts before and after.

## Constraints

- Never remove an item whose delivery is not represented in the durable record — the
  changelog or tags for a version, the spike document for a `SPIKE:` marker.
- Never report a spike-completed item as a changelog gap, and never interpret a legacy
  free-text completion value as though it were a version.
- Never remove an item with remaining work, whatever its status says.
- Never prune at all when `<vcs.system>` is `none` and `<paths.changelog>` is null.
- Do not touch `<paths.work>/active-release.md` or `<paths.work>/project.yml`.
- Do not run mutating git commands. Under `git`, the human reviews the diff; under `none`,
  say plainly that the removal is immediate and unrecoverable, and get confirmation on that
  basis.
- Do not edit `<paths.changelog>`; report gaps instead of filling them from a pruned item's
  summary.
