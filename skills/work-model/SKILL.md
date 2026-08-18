---
name: work-model
description: >
  Explains and enforces the project-agnostic Request -> Work item -> Active release -> Done
  work-management model, including the manifest schema at docs/work/project.yml, the exact
  file formats for requests.md / backlog.yml / active-release.md, all valid statuses, and the
  configuration-resolution protocol every command follows. Use when someone asks how the work
  management model works, what a status means, whether something is blocked or deferred, what
  fields a request or work item takes, how to configure project.yml, what belongs in the
  manifest versus a command, or when validating that requests.md and backlog.yml conform to
  the model. Also use before hand-editing any of the work files, so the edit matches the
  required structure rather than drifting from it.
---

# Work model

Authoritative reference for the work-management model and its manifest. Consult this
before answering a question about the model, before hand-editing any work file, and
before adding a project fact anywhere other than the manifest.

## Resolve configuration first

Read `references/config-resolution.md` and follow it. Project facts come from
`<paths.work>/project.yml` and nowhere else. Report a missing manifest rather than
guessing paths.

That document also defines **the two roots** — the VCS root where `.git` lives, and the
project root where this project's world begins — and the path-resolution and VCS-scoping
rules that follow from telling them apart. Read that section before resolving any path or
running any git command; on a repository holding one project the two roots coincide and
nothing changes, but the assumption that they always coincide is wrong on a monorepo.

**The supported schema is `model_version: 5`.** The compatibility gate in
`references/config-resolution.md` is checked before any other field, by every command. A stale
manifest is a stop with a pointer to `/work-init --upgrade`, never an auto-migration.

`references/schema-history.md` holds every earlier era, the keys that identify it, and its
transition forward. Consult it when asked what changed between versions, or when a manifest does
not match its own declaration — `model_version: 2` covers four distinct shapes, so those files
must be identified by their keys rather than their declaration.

## The model

Read `references/model.md` for the full model: the four files, the request and work item
field schemas, all three status vocabularies, spike rules, the active release lifecycle,
and the pruning rules.

Answer questions about statuses, fields, and lifecycle from that document rather than
from general project-management knowledge. The vocabularies are deliberately specific —
`blocked` versus `deferred`, and `abandoned` versus `cancelled`, both carry precise
meanings that a plausible-sounding paraphrase will get wrong.

## The dividing line

When something looks like it needs recording, decide where it belongs:

| The fact is... | It belongs in |
|---|---|
| true of this project only (paths, boundary, stack, agents, release mechanics, test policy) | the manifest, `project.yml` |
| true of the model on every project (statuses, field schemas, lifecycle) | `references/model.md` — do not edit per project |
| a change to the manifest schema itself | `references/schema-history.md`, plus a `model_version` bump |
| a piece of work someone wants done | `requests.md`, via the `work-capture` skill |
| refined, actionable, ready for an implementer | `backlog.yml`, via `/work-plan` |
| the current delivery scope | `active-release.md`, via `/work-plan` |
| a durable record of something shipped | the changelog, git history, tags, tests, code |

If a command or an agent file would have to change to record a project fact, that is the
signal the fact belongs in the manifest instead.

## Validating conformance

When asked to check the work files, or before a planning session on a project last
touched by hand, verify:

- **Structure** — `requests.md` has the legend and the four H2 sections in order; each
  request is an H3 with `Key: value` lines, no bullets or nested headings.
- **Vocabulary** — every `Type:` appears in `<taxonomy.request_types>`; every work item
  `type` appears in `<taxonomy.work_types>`; every `Priority:` in
  `<taxonomy.priorities>`; every status is valid for its plane.
- **Field names** — requests use `Work items:` (not `Derived work items:`).
- **Paired fields** — `blocked` carries `Blocked on:` / `blocked_on`; `deferred` does
  not; `done` and `partially-done` carry `Done in:` / `done_in`.
- **Completion values** — every entry in `Done in:` / `done_in` is a release version or
  `SPIKE: <ITEM-ID>`. Free text such as a bare date is drift: a spike bumps no version, so
  it records the marker instead.
- **Referential integrity** — every ID in `Work items:` exists in `backlog.yml`; every
  `source_request` exists in `requests.md`; every name in `suggested_agents` exists in
  `<agents>` and not in `<inactive_agents>`; every `dependencies` entry resolves.
- **Ownership** — compute effective ownership as `owns` minus `excludes`, expanding globs and
  treating a parent path as covering its children. No path may then belong to two agents.
  This is a hard failure, not a warning: overlapping ownership generates two contradictory
  scope tables that each tell the other agent to keep out, and the deadlock is hard to trace
  back to its cause. A carve-out must be declared on both sides — excluded by one agent,
  owned by another — never left to a most-specific-path-wins convention, because nothing
  downstream enforces one. A path excluded by one agent and owned by none is a gap. Confirm
  every `owns`, `excludes`, and `reads` path resolves against the tree; a typo produces an
  agent that owns nothing, or hands a path back to the wrong owner.
- **Boundary coherence** — resolve every path field against the project root per
  `references/config-resolution.md`, then confirm each one exists. A path that resolved
  correctly under the old repository-root rule and is now missing is the signature of a
  manifest that kept an `apps/toolA/` prefix it no longer needs; say so rather than reporting
  five unrelated missing files. Where `<scope.root>` is set, confirm it names a directory that
  exists and sits inside the repository, and that every leading-slash path in any agent's
  `owns` also appears in `<scope.writes_outside>` — an owned path outside the boundary that
  nothing has authorised is a write the boundary will refuse at the moment it is attempted.
  Confirm each `<scope.writes_outside>` entry resolves and sits outside `<scope.root>`; an
  entry that is already inside the boundary is redundant and reads as a permission that was
  needed once.
- **Tag coherence** — where `<scope.root>` is set and `<version.tag_template>` is null, flag
  it: the project is claiming the repository's global `v1.2.3` and will collide with the next
  member to reach that number. Where the template is set and
  `<release.pipeline.definition>` names a file with `trigger: tag-push`, read that file and
  confirm its tag filter actually matches what the template renders. A workflow watching `v*`
  does not fire on `toolA/v1.2.3`, and that combination ships a release that builds nothing.
- **Stage coherence** — check `<vcs.stages>` against the validity table in
  `references/config-resolution.md`, and report every failure at once rather than the first.
  The rules that catch real misconfigurations: `commit: human` with any later stage set to
  `agent` (nothing downstream works without commits); `merge` and `pull_request` both active;
  either active without `branch`; `pull_request` without `push`; `tag: none` with
  `<version.file>` null, which leaves the version nowhere to live. Also flag interleaved
  ownership — an `agent` stage between two `human` ones — not as an error but as a session
  that will stall mid-flight, which the human should know before a release rather than during
  one.
- **Version-control coherence** — if `<vcs.system>` is `none`, then
  `<version.file>` must be set, because a tag is not available to hold the version — and
  `<version.tag_template>` is inert, since there are no tags to name;
  `<paths.changelog>` should be set, because it is the only remaining narrative record; and
  every stage in `<vcs.stages>` must be `none`, since there is no repository to act on. If
  `<vcs.system>` is `git`, confirm a repository actually exists — a manifest claiming git on
  a directory with no repository will make every release hand off to steps the user cannot
  perform. Test with `git rev-parse --show-toplevel`, not by looking for a `.git` directory
  beside the manifest: on a monorepo member `.git` is legitimately several levels above, and
  a bare directory check reports no repository on a project that has one.
- **Hygiene** — flag if `done` items older than the most recent few releases are still
  present, which means pruning has lapsed. Suggest `/work-prune`.
- **Schema** — `model_version` matches the supported version, *and* the file's shape agrees with
  its own declaration. A mismatch either way is a stop; the fix is `/work-init --upgrade`, which
  migrates the manifest and regenerates the agents in one pass.

Report findings as a list with a recommended fix for each. Do not silently repair
vocabulary drift: if `requests.md` uses a type the manifest does not list, the manifest
may be wrong rather than the data. Ask which is authoritative.

## Related commands

`/work-init` sets a project up, and `/work-init --upgrade` brings an older manifest to the
current schema before regenerating agents — it runs the `/work-migrate` procedure itself, which
also stays callable on its own when you want that edit reviewed separately. `/work-ingest`
backfills intake from a PRD or other existing material. `/work-plan`
refines and proposes scope. `/work-release` delivers. `/work-spike` runs investigations.
`/work-crunch` runs those three on a loop under a permissions contract agreed up front.
`/work-review-deferred` re-triages parked items. `/work-prune` trims completed history. The
`work-capture` skill handles quick intake.

On a project adopting the model with work already recorded elsewhere, the order is
`/work-init` then `/work-ingest` then `/work-plan`. Ingesting before the manifest exists has no
vocabulary to validate against; refining before ingesting means refining an empty file.
