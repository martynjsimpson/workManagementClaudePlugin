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

## A project is not always a repository

The model manages **a project**, which may be a whole repository or one member of a larger
one — an app in a monorepo, a package in a workspace. `<scope.root>` says which: null for a
whole repository, a path like `apps/toolA` for a member.

Two roots follow from that, and they are not the same thing:

- the **VCS root**, where `.git` lives. Branches, tags, pushes and merges are repository-wide
  by nature and happen here.
- the **project root**, where this project's world begins and ends. Every path in the manifest
  resolves against it, and nothing writes outside it.

Where a project is a whole repository the two coincide, which is why the distinction can go
unnoticed for a long time and then matter all at once. `config-resolution.md` holds the
resolution rules, the write boundary, and the git scoping that follows; read it before
resolving a path or running a git command.

The consequence for the model itself is small but real: **the durable record is repository-wide
while the work is not.** A tag, a branch name and a commit all live in a namespace shared with
every other member, so each must carry something that identifies this project — which is why
release branches are slugged and `<version.tag_template>` exists.

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
Parked since: <version or date the item first stalled — see "Review pressure">
Reviewed: <count> — last <version or date>, <one line on why it is still open>
```

Do not invent additional fields. Do not use sub-headings inside a request block.

### Write a request the human can read

A request is the one file in this model addressed to a person. It is read by whoever
decides what happens next, which is usually someone who has not read the code the request
came out of — so `Title:` and `Summary:` are written for that reader, not for the agent
that will implement it.

- **`Title:` states the effect, not the mechanism.** "Security guidance is written out
  twice — keep both copies or cite one?" rather than "Decide whether C4's
  guidance-placement rule should be de-duplicated between the standard and the update
  skill". A title that cannot be understood without opening two other files is a title
  that will be skipped.
- **`Summary:` must stand alone.** Rule ids, file paths, symbol names and item IDs may
  appear, but only after the sentence that says what is at stake without them.
- **A request that asks the human a question ends its `Summary:` with that question.**
  Not in `Notes:`, and not under a heading partway down it. `Notes:` is background for
  whoever picks the work up; a question buried there has not been asked.
- **Mechanism, evidence and history go in `Notes:`.** Nothing is lost by moving them —
  they are simply below the part that has to be read.

This applies to every request whoever writes it, and it binds hardest on requests written
by an agent, because those are the ones drafted in the voice of the session that found the
problem rather than the voice of the person who has to decide about it.

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
  source_request: <request_prefix>-001   # or null — set when a human asked for this
  source_release: <version>              # or null — set when a release found it
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
  parked_since: <version>                # when it first stalled — see "Review pressure"
  reviewed: <count>                      # times re-checked without changing
```

### Where a work item came from

`source_request` and `source_release` record provenance, and **exactly one of them is
set**. A work item traces either to something a person asked for or to something a
release discovered — never both, never neither.

- `source_request: <request_prefix>-nnn` — a human-facing ask was refined into this.
- `source_release: <version>` — a release surfaced this while building something else,
  and it needed no human decision to specify. There is no request, deliberately.

They are two keys rather than one free-text field because the difference is the only
cheap way to answer a question worth asking regularly: **every work item carrying a
`source_release` and no `source_request` is work that nobody asked for.** Most of it is
legitimate upkeep on code that already exists. Some of it is scope arriving through the
back door. Collapsing both origins into one field makes the two indistinguishable, and
the question then costs a full read of the backlog instead of a grep.

`evidence` is a different field and does not substitute for either: it points at the spec
section or file that justifies the work, not at who wanted it.

**Both fields are additive, and older backlogs predate `source_release`.** An existing item
carrying `source_request: null` and nothing else is legacy, not invalid. Backfill it when you
next touch that item for another reason, from its `evidence` or the release it shipped in;
do not sweep the file to fix them all, and do not block on one. New items written from this
version on always set exactly one.

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

## Review pressure

Some items are re-checked every planning session — `blocked` and `partially-done`
requests, `blocked` work items — and `deferred` items are re-checked whenever
`/work-review-deferred` runs. An item that survives many of those checks unchanged is not
a neutral fact about the backlog. **It is a decision nobody has made**, and the longer it
sits the more likely the right answer is to drop it rather than to keep carrying it.

Two fields make that visible, on requests as `Parked since:` / `Reviewed:` and on work
items as `parked_since` / `reviewed`:

```text
Parked since: 0.18.4
Reviewed: 7 — last 0.22.0, still waiting on a risk call only the human can make
```

- `Parked since:` is written once, when the item first stalls, and never changed after.
- `Reviewed:` is **replaced** at every re-check, with the count incremented by one.

### Replace the review note; do not append to it

A re-check that finds nothing changed **overwrites** the `Reviewed:` line and adds
nothing to `Notes:`. It does not append a dated paragraph restating the item.

The rule of thumb: *if the note would read the same as last time with a new date on it, it
replaces rather than appends.*

This matters more than it looks. A restatement per session turns `Notes:` into a
transcript that grows without bound, and the transcript is storage for what is really a
counter — so the file pays hundreds of lines to hold an integer. These files are version
controlled; the history of a request is its commit history, and duplicating that history
inside the working copy makes the live file harder to read without making the record any
more durable.

**The carve-out is a material change.** Append to `Notes:` when the situation actually
moved — the question changed shape, a new constraint appeared, the human answered with
something other than yes or no, an adjacent decision overtook it. Those are information.
"Unchanged, still waiting" is not, and it belongs in the `Reviewed:` line where one copy
of it lives.

### Escalation at five

Once `Reviewed:` reaches **5**, restating the item is no longer an acceptable outcome of a
review. The next review must stop and put three options to the human explicitly:

1. **Do it now** — promote it into the next release scope.
2. **Reject it** — it has been carried five times without anyone wanting it enough to act.
3. **Park it with a named trigger** — a specific thing that would make it live again. Where
   that trigger is a dependency, the item is `blocked` and the trigger goes in
   `Blocked on:` / `blocked_on`; where it is a condition rather than a dependency, it goes
   in `Notes:` and the `Reviewed:` count resets to 0, because the item is now waiting on
   something stated rather than on nobody having decided.

**Reject must be offered as a first-class option, in those words.** An item reviewed five
times is evidence about how much it is wanted, and the escalation exists to convert that
evidence into a decision. An escalation that only ever offers "do it" or "keep waiting"
asks the same question a sixth time.

Report the count when you escalate — "this has been reviewed five times since 0.18.4" is
the whole point, and it is the part the human is being asked to weigh.

## Spike work items

A spike is investigation-only work. It produces knowledge, not shippable code.

Use `type: spike` when the correct implementation approach is unknown and must be
researched before a feature can be scoped, or when a cross-cutting concern needs to be
explored and documented before work items can be written.

**Output:** a document at `<paths.spikes>/<ITEM-ID>.md` containing at minimum:

- `## Findings` — what was discovered.
- `## Recommendations` — specific, actionable next steps, phrased so follow-on backlog
  items can be written directly from them. Not high-level observations.

**Recommendations are numbered `R1`, `R2`, … and each number is one recommendation.** The
number is the unit of triage: what becomes a work item, a request, or an explicit decline
when the spike's release closes out. A recommendation may carry sub-points beneath it —
cited as `R6 item 5` — but those are detail within one recommendation, not separate
recommendations, and they are not triaged individually.

Numbers are stable once written. A later session citing `R4` must find the same
recommendation there, so recommendations are never renumbered and a superseded one is
marked rather than removed. This is what lets `evidence` point at
`<paths.spikes>/<ITEM-ID>.md R4` and stay true.

**Done for a spike** means that document exists with both sections populated. No code
ships. No version is bumped. No test coverage is expected.

**The recommendations are the deliverable, and they are processed when the spike's release
closes out** — not left in the document for someone to find. Work derived from them
**inherits the spike work item's own provenance**: the `source_request` that commissioned
the investigation, or its `source_release` where the spike itself came from a release
finding. A spike is a route between an ask and the work it leads to, not a third kind of
origin, which is why there is no `source_spike` key. The link to the document is carried by
`evidence`, as `<paths.spikes>/<ITEM-ID>.md` plus the section it came from — a mention in
prose is not a reference anything can follow.

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
all is set by `<vcs.system>`, and who performs each release stage by `<vcs.stages>`.

**There is no global "who owns version control" answer.** A release performs six things —
branch, commit, push, merge or pull request, and tag — and each says separately whether it
happens and who does it: `none`, `agent`, or `human`. Cutting the branch yourself while the
agent commits and opens the pull request is an ordinary configuration, not a special case.
`config-resolution.md` holds the stage order, the vocabulary, and the validity rules.

**Versioning and version control are independent.** A project may carry a version in
`package.json` with no repository, or sit under git with the version expressed only as a tag.
Neither implies the other, which is why they are separate manifest blocks.

**The version is assigned at the start of a release, not at the end.** `/work-plan` always
writes `Version: TBD`; `/work-release` confirms the number immediately after scope approval and
writes it to `<version.file>` and `<version.mirrors>` before any implementation begins, so
everything built during the session carries the number it will ship as. The consequence is that
an abandoned release has already bumped the version, and restoring it is part of the
abandonment rather than optional tidying. Under `git` with `<vcs.stages.commit>` `agent` the
bump is committed on its own, immediately and before any agent is spawned, so it is the first
release-owned commit on the branch: the anchor a rollback rewinds to, and a single revert
rather than a hand-restore of every versioned path.

**The release coordinator is a persona, not an agent.** It is the role `/work-release`
adopts for the duration of a release session. There is no `release-manager` or
`release-coordinator` sub-agent on any project, and one must never be spawned. When
a stage is owned by `agent`, it is that persona performing it. The stage value `agent` names
who acts — the AI rather than the human — and never authorises spawning a sub-agent for it.

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

Where `<vcs.system>` is `git` and `<vcs.stages.commit>` is `agent`, that interface extends to the
repository: the delivery command commits `active-release.md`, `backlog.yml` and
`requests.md` as they change, starting with a commit of the planning state once the release
branch exists and continuing through each status transition. An uncommitted status line has
not moved for anything reading the repository rather than the working tree. Where
`<vcs.stages.commit>` is `human`, the command commits nothing and names those files in its
handoff instead — which, because commit is the pivot every later stage depends on, means the
whole release is a handoff.

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
   Provenance is likewise its own field: exactly one of `source_request` / `source_release`
   on every work item.
6. Version assignment follows `<version.owner>`; each version-control action follows its own
   entry in `<vcs.stages>`. These are independent, and none is decided per session.
7. Nothing is written outside the project root unless `<scope.writes_outside>` authorises
   that exact path. Reads are unrestricted.
8. If a work item is uncertain, its first action is audit or clarification, not
   implementation.
9. Keep the model small.
