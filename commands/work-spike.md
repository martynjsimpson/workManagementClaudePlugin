---
description: Run the spike work items in the active release — investigation only, producing one Findings/Recommendations document per spike.
---

Coordinate a spike session. Spikes produce investigation documents only. No code ships, no
version is bumped, no release pipeline is triggered.

## Step 0 — Resolve configuration

Follow `${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/config-resolution.md`. Read the
manifest at `<paths.work>/project.yml`. Refuse to proceed if it is missing.

## Step 1 — Read scope and safety-check

Read `<paths.work>/active-release.md` for the selected work items, then look up each
item's `type` in `<paths.work>/backlog.yml`.

**Stop if any selected item is not a spike.** Report:

> This release contains non-spike work items: [list]. `/work-spike` only handles spikes.
> Run `/work-release` instead, or ask the PM to split the spike and non-spike items into
> separate releases.

If every item is a spike, list them (ID, title, one-line question) and ask whether to run
all of them or specific ones. Wait for confirmation.

## Step 2 — Run each spike sequentially

**Set `Status: in-progress` in `<paths.work>/active-release.md` before assigning the first
spike.** A spike release moves through the same status line as a code release, and this is
the only signal outside this session that the investigation started: anything watching the
file — a human, a monitoring tool, or `/work-crunch` resuming an interrupted run — sees a
release still sitting at `approved` as one where nothing has begun, and treats re-running
the spikes from the top as safe. Commit that change where `<vcs.system>` is `git` and
`<vcs.owner>` is `command`, per Step 3.

Finish each spike completely before starting the next. Spike findings frequently change
what the next spike should even ask.

For each spike:

**Assign it.** Use the item's `suggested_agents`, or the architect from `<agents>` if the
spike concerns a cross-cutting or architectural question. Only names in `<agents>`.

**Brief it** with:

- the spike ID and title;
- the full investigation question, and every specific sub-question the release records;
- the output path: `<paths.spikes>/<ITEM-ID>.md`;
- the required structure — a `## Findings` section and a `## Recommendations` section;
- the standard for recommendations: specific enough that the PM can write follow-on
  backlog items directly from them. Not observations, not "further investigation is
  warranted" — actionable next steps, and where a decision is needed, the options with a
  recommendation.

**Verify the output** before accepting it. Confirm the file exists, that `## Findings` has
substantive content, and that `## Recommendations` is actionable. If a recommendation
could not be turned into a backlog item as written, send it back. A thin spike document is
worse than none, because it looks like the question was answered.

**Mark it done** in `<paths.work>/active-release.md`, then move on. That is the work item's
own status line, not the release's — the top-level `Status` stays `in-progress` until Step 3.

## Step 3 — Wrap up

**Set the top-level `Status` in `<paths.work>/active-release.md` to `ready-for-release`.**
Do this first, before reporting anything. It is the step that ends the release: marking
every work item done leaves the release itself still open, `/work-plan`'s closeout runs on
the top-level status alone and does nothing without it, and a release stuck at `approved`
or `in-progress` is one a later session will offer to run again.

Tell the human which documents were produced and where, then: review the Findings and
Recommendations in each, and run `/work-plan` to process the recommendations into requests
and backlog items.

Handle the spike documents per `<vcs>`. When `<vcs.system>` is `none`, there is nothing to
commit — just name the files written. When it is `git` and `<vcs.owner>` is `human`, note
that the documents are uncommitted files for them to commit as normal, along with
`<paths.work>/active-release.md`. When `<vcs.owner>` is `command`, commit them: spike
documents are the deliverable, and the same ownership rule applies to them as to code.

Where you commit, commit `active-release.md` too, and commit each status transition as it
happens rather than only here — `in-progress` in Step 2, `ready-for-release` above. An
uncommitted status line has not moved as far as the repository is concerned, and a spike
release is the case most likely to be read from outside the session, because there is no
version bump or tag to signal it instead. Name the paths explicitly; never sweep the tree
with a repo-wide `git add`. Where `<scope.root>` is set, `git add -A`, `git add .` and
`git commit -a` are forbidden outright, and every path staged must resolve under the project
root — a spike document written outside the boundary is a spike investigating someone else's
app.

## Constraints

- Never leave the release at `approved` once the first spike has been assigned, or at
  `in-progress` once the last document has been accepted. Both transitions are required
  even where the session is going to report the outcome in chat anyway — the status line
  is what anything outside this session reads.
- Do not write application code. A spike that produces code is not a spike.
- Do not bump a version or update `<paths.changelog>`.
- Do not edit `<paths.work>/backlog.yml` or `<paths.work>/requests.md` — `/work-plan`
  closes those out.
- Do not edit `<paths.work>/project.yml`.
- Do not accept a document missing either required section.
