# The Work Management Model

A small, repo-local delivery model for AI-supported development. It exists to answer
four questions and nothing more:

1. What have we noticed, wanted, or asked for?
2. What actionable work has been refined from those asks?
3. What are we actively building and releasing now?
4. What has shipped?

```text
Request -> Work item -> Active release -> Done
```

There are no slices, batches, phases, or side-car planning documents. A selected group
of work items **is** the active release. The model is intentionally small so it can be
replaced by an issue tracker later without losing anything.

## Project-agnostic by construction

Every project-specific fact — name, paths, ID prefixes, type vocabulary, agent roster,
release mechanics, test policy — lives in one manifest at `<paths.work>/project.yml`.
This document describes only the parts that are the same on every project.

Values written as `<key>` below are manifest lookups. Never substitute a literal path,
project name, or agent name from memory.

## Files

```text
<paths.work>/
  project.yml         <- the manifest; the only project-specific file
  README.md           <- operating guide for humans and agents
  requests.md         <- human-friendly intake
  backlog.yml         <- refined, agent-readable work items
  active-release.md   <- the single current delivery scope

<paths.spikes>/
  <ITEM-ID>.md        <- one file per spike work item
```

Agents must not create new planning files, phase files, or side-car backlogs without
human approval. Agents must not read superseded planning documents by default.

## Requests

A request is the human-facing capture of something wanted or noticed. Requests may be
rough; they do not need to be implementation-ready. The human should be able to capture
one without thinking about tickets, agents, releases, or architecture.

`requests.md` has a fixed structure:

1. **Legend** — the valid statuses and what each means. Always at the top.
2. **Four H2 sections, in this order:**
   - `## Inbox / needs refinement`
   - `## Refined requests`
   - `## Done`
   - `## Deferred / rejected`
3. **Each request** is an H3 heading carrying the request ID, followed by plain
   `Key: value` lines — not bullets, not a table, not YAML frontmatter.

```text
### <request_prefix>-001
Request ID: <request_prefix>-001
Title: <short title>
Type: <one of taxonomy.request_types>
Status: <one of the request statuses below>
Priority: <one of taxonomy.priorities>
Summary: <one or more sentences describing the ask>
Notes: <optional — only when there is something worth noting>
Work items: <backlog item IDs once they exist>
Source: <where it came from, e.g. "human request (direct)">
Done in: <release version, or SPIKE: <ITEM-ID> — only when status is done or partially-done>
Blocked on: <named dependency — only when status is blocked>
```

Do not invent additional fields. Do not use sub-headings inside a request block.

### Request statuses

| Status | Meaning |
|---|---|
| `inbox` | Captured, not yet reviewed. |
| `needs-refinement` | Reviewed, but needs clarification, splitting, or human input before reliable work items exist. |
| `refined` | One or more work items exist, but none are currently selected for delivery. |
| `in-active-release` | One or more derived work items are selected in `active-release.md`. |
| `partially-done` | Some of the outcome has shipped, but meaningful scope remains. Reviewed every planning session — see below. |
| `done` | The intended outcome is satisfied. |
| `blocked` | Cannot proceed until a named dependency resolves. **Must** carry `Blocked on:`. Reviewed every planning session. |
| `deferred` | Valid but deliberately parked with no specific dependency. **Not** surfaced automatically; reviewed only on request. |
| `rejected` | Out of scope or deliberately not pursued. |
| `duplicate` | Covered by another request or work item. |

`blocked` and `deferred` are not interchangeable. If there is a specific thing that has
to happen first, it is `blocked`, and the thing must be named.

A `partially-done` request must record what remains in `Notes:` — otherwise the
remaining scope has nowhere to live and gets lost. `/work-plan` checks every
`partially-done` request each session and drafts a fresh `## Inbox / needs refinement`
request for the remaining scope once it is not already covered by an existing request
or work item; nothing else revisits it. A `partially-done` request belongs in `##
Refined requests` — filing it under `## Done` hides it from that check.

## Work items

A work item is an actionable, refined unit of work in `backlog.yml`. It must be clear
enough that an implementer can start from it without reading any historical planning
document.

The file begins with `model_version: 1` and contains an `items:` list.

```yaml
- id: <work_prefix>-001
  source_request: <request_prefix>-001   # or null
  title: short description
  type: <one of taxonomy.work_types>
  capability: product area or capability
  status: <one of the work item statuses below>
  priority: <one of taxonomy.priorities>
  confidence: high | medium | low
  summary: multi-sentence description of the work
  acceptance:                            # testable criteria
    - ...
  remaining: what is still to do
  dependencies: []                       # work item IDs
  suggested_agents: []                   # names from manifest agents[]
  evidence: reference to the spec section, file, or note justifying this item
  done_in: []                            # only when status is done — versions, or SPIKE: <ITEM-ID>
  blocked_on: ...                        # only when status is blocked
```

### Work item statuses

```text
needs-refinement   ready   needs-audit   shippable-candidate
in-progress        needs-test           blocked   done   deferred
```

- `blocked` **requires** a `blocked_on` field naming the dependency.
- `deferred` must **not** carry `blocked_on`. If there is a dependency, use `blocked`.
- `done` carries completion metadata in `done_in`, never encoded in the status value.

## Status boundaries

Three status planes, deliberately independent:

- **Request status** reflects what happened to the human-facing ask.
- **Work item status** reflects what happened to the refined implementation work.
- **Active release status** reflects what is currently being built and released.

Do not try to make request status mirror every downstream work item status. A request
with three work items, one shipped and two outstanding, is `partially-done` — not three
statuses at once.

## Spike work items

A spike is investigation-only work. It produces knowledge, not shippable code.

Use `type: spike` when the correct implementation approach is unknown and must be
researched before a feature can be scoped, or when a cross-cutting concern needs to be
explored and documented before work items can be written.

**Output:** a document at `<paths.spikes>/<ITEM-ID>.md` containing at minimum:

- `## Findings` — what was discovered.
- `## Recommendations` — specific, actionable next steps, phrased so follow-on backlog
  items can be written directly from them. Not high-level observations.

**Done for a spike** means that document exists with both sections populated. No code
ships. No version is bumped. No test coverage is expected.

**Completion is recorded as `SPIKE: <ITEM-ID>`**, in the work item's `done_in` and in the
`Done in:` of any request it completes — never a version, and never free text. A spike bumps
no version, so there is no number to record; without a fixed form, what lands in a field
every other item parses as a version is improvised prose like
`spike completed 2026-08-03 (no release/code change)`, which is neither comparable to a
cutoff nor a pointer to anything.

The ID is the whole value. The document is at `<paths.spikes>/<ITEM-ID>.md` by construction,
so the path is derived when needed rather than stored — a stored path goes stale the first
time `<paths.spikes>` moves, whereas the ID stays true.

A `done_in` therefore holds versions or spike markers, and may hold both: a request satisfied
by an investigation and then an implementation legitimately carries
`SPIKE: <ITEM-ID>` alongside a version.

**Ownership exception.** Whichever agent is assigned a spike may write its document, even
where `<paths.spikes>` sits in another agent's `owns` list. Spike output is a defined
deliverable of the work item, not a reach across an ownership boundary. Without this
exception a spike assigned to a developer would be unwritable by the only agent tasked
with it.

## Active release

The selected group of work items for the current delivery cycle, in
`active-release.md`. It is not a second backlog.

It contains: the release goal, selected work item IDs with their status, out-of-scope
items, decisions made and decisions needed, required agents, blockers, verification
bar, and the version once known.

A release may open as:

```text
Version: TBD
Status: proposed
```

Who assigns the version is set by `<version.owner>`. Whether there is any version control at
all is set by `<vcs.system>`, and who operates it by `<vcs.owner>`.

**Versioning and version control are independent.** A project may carry a version in
`package.json` with no repository, or sit under git with the version expressed only as a tag.
Neither implies the other, which is why they are separate manifest blocks.

**The version is assigned at the start of a release, not at the end.** `/work-plan` always
writes `Version: TBD`; `/work-release` confirms the number immediately after scope approval and
writes it to `<version.file>` and `<version.mirrors>` before any implementation begins, so
everything built during the session carries the number it will ship as. The consequence is that
an abandoned release has already bumped the version, and restoring it is part of the
abandonment rather than optional tidying. Under `git` with `<vcs.owner>` `command` the bump
is committed on its own, immediately and before any agent is spawned, so it is the first
release-owned commit on the branch: the anchor a rollback rewinds to, and a single revert
rather than a hand-restore of every versioned path.

**The release coordinator is a persona, not an agent.** It is the role `/work-release`
adopts for the duration of a release session. There is no `release-manager` or
`release-coordinator` sub-agent on any project, and one must never be spawned. When
`<vcs.owner>` is `command`, it is that persona performing the commit, tag, and push.

**A project need not be under version control.** When `<vcs.system>` is `none`, every VCS
step is skipped rather than failed: no commit, no tag, no branch, no working-tree check, and
no instruction to the human to perform one. The version is written to
`<version.file>` instead of expressed as a tag. This is a supported configuration,
not a degraded one — but see the consequences for the durable record below.

### Active release statuses

| Status | Meaning |
|---|---|
| `none` | No release in flight. |
| `proposed` | Scope drafted, awaiting human approval. |
| `approved` | Scope approved, implementation not started. Written by whoever takes the approval — `/work-plan` when the human approves the proposal in that session, `/work-crunch` when its pre-flight contract approves it — never by a delivery command. |
| `in-progress` | Implementation under way — or, on a spike release, investigation under way. |
| `testing` | Implementation complete, verification under way. Not used by a spike release, which has no separate verification phase. |
| `ready-for-release` | Verified, awaiting the release action. |
| `released` | Shipped, with the version recorded. |
| `abandoned` | Started but not completed, **and all code changes reverted**. Requires an `## Abandonment note`. |
| `cancelled` | Scope withdrawn **before implementation started**. No code was written, so nothing needs reverting. |

**The status line is the release's external interface.** It is the only record of where a
release has got to that survives outside the session running it, and the only thing
`/work-plan`'s closeout and `/work-crunch`'s entry point read. So every delivery command
moves it as it goes, and a spike release is no exception: `/work-spike` sets `in-progress`
before the first spike is assigned and `ready-for-release` once the last document is
accepted, giving `approved → in-progress → ready-for-release` where a code release gives
`approved → in-progress → testing → ready-for-release`. Marking individual work items done
is not a substitute — nothing downstream reads them to infer that the release finished.

Where `<vcs.system>` is `git` and `<vcs.owner>` is `command`, that interface extends to the
repository: the delivery command commits `active-release.md`, `backlog.yml` and
`requests.md` as they change, starting with a commit of the planning state once the release
branch exists and continuing through each status transition. An uncommitted status line has
not moved for anything reading the repository rather than the working tree. Where
`<vcs.owner>` is `human`, the command commits nothing and names those files in its handoff
instead.

The difference between `abandoned` and `cancelled` is whether code was written and
undone. An abandonment note must state why, what was not delivered, what must happen
before re-proposing, and any manual cleanup required (stale changelog entries, database
state). It is processed at the start of the next planning session.

## Completion and pruning

This model is an operating model for live work, not a historical archive. Completed items
are pruned once their outcome is recorded somewhere durable.

**What counts as the durable record depends on `<vcs.system>`.**

Under `git`, it is the changelog, commit history, release tags, tests, code, and decision
records — several independent copies, so pruning is low-risk.

Under `none`, it is the changelog and `<version.file>`, and nothing else. There is
no commit history to fall back on. Pruning is therefore genuinely destructive rather than
merely tidy: once an item is removed from `requests.md`, if it was not written into the
changelog, the information is gone. Prune conservatively, and never without confirming the
changelog entry exists first.

Completed requests and work items may remain briefly after a release, then should be
pruned. Prune when:

- a request has `Status: done` and all `Done in:` versions are older than the cutoff;
- a work item has `status: done` and all `done_in` versions are older than the cutoff;
- the item has no unresolved remaining work.

**A `SPIKE: <ITEM-ID>` marker is not a version and is not age-gated.** There is no number to
compare against a cutoff, and the durable record is the spike document rather than the
changelog. An item completed by spike is prunable once that document exists at
`<paths.spikes>/<ITEM-ID>.md` with both required sections — subject to the same dependency
and referential checks as anything else. Do not report its missing changelog entry as a gap:
a spike correctly has none.

Never prune `partially-done`, `deferred`, `blocked`, `needs-audit`,
`needs-refinement`, `ready`, or `in-active-release` items.

If `requests.md` has grown past a few hundred lines of `done` entries, pruning has
lapsed. That is a maintenance signal, not a normal state.

## Rules

1. `requests.md` is for rough human intake.
2. `backlog.yml` is for refined, agent-readable work.
3. `active-release.md` is the only current delivery scope.
4. There is no batch or slice concept.
5. Completion metadata is separate from status (`Status: done` + `Done in:`).
6. Version assignment follows `<version.owner>`; version-control actions follow
   `<vcs.system>` and `<vcs.owner>`. These are independent, and none is decided per session.
7. If a work item is uncertain, its first action is audit or clarification, not
   implementation.
8. Keep the model small.
