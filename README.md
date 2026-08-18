# work-management

A Claude Code plugin that gives a repository a small, explicit delivery model —
`Request -> Work item -> Active release -> Done` — plus the commands that plan against it,
ship from it, and close it out.

The model is small on purpose: four work files in your repo, three independent status planes,
and enough structure that agents can deliver from it without a planning-document sprawl. What
makes it portable is that **every project-specific fact lives in one manifest**. The commands,
the skills, and the model document contain no paths, no project names, no agent names, and no
release mechanics.

## The model

```text
Request  ->  Work item  ->  Active release  ->  Done
```

- **Request** — something wanted or noticed, captured in your own words. Rough is fine.
- **Work item** — a refined, agent-ready unit with acceptance criteria.
- **Active release** — the selected work items, being built now.
- **Done** — the durable record: changelog, tags, tests, and the code itself.

It answers four questions and nothing else: what have we asked for, what actionable work came
out of it, what are we building now, and what has shipped. A selected group of work items *is*
the active release — there are no slices, phases, or side-car planning documents.

The manifest and the four work files sit together in the repo:

```text
docs/work/                # location is configurable
  project.yml             the manifest — the only project-specific file
  requests.md             human-friendly intake
  backlog.yml             refined, agent-readable work items
  active-release.md       the single current delivery scope
  README.md               operating guide for humans and agents
```

Statuses are **not** configurable — they are the model. They sit in three deliberately
independent planes: a **request** status (what happened to the human-facing ask), a **work
item** status (what happened to the refined implementation work), and an **active release**
status (what is being built right now). A request with one work item shipped and two
outstanding is `partially-done`, not three statuses at once.

## The three layers

| Layer | Where it lives | Changes when |
|---|---|---|
| Engine — commands, skills, templates | this plugin | you improve the model |
| Manifest — every project fact | `docs/work/project.yml` in the repo | you set up or change a project |
| Content — requests, backlog, release, agents | the repo | you do the work |

Update the plugin once and every project benefits. Adding a project fact never means editing
a command.

## Install

Add this repo as a marketplace, then install the plugin:

```text
/plugin marketplace add martynjsimpson/workManagementClaudePlugin
/plugin install work-management@work-management
```

Updates ship by pushing a new `version` to `.claude-plugin/plugin.json` — enable
auto-update for this marketplace (`/plugin` → **Marketplaces** → **Enable auto-update**)
to pick them up automatically, or run `/plugin marketplace update` and `/plugin update`
by hand.

## Set up a repository

In the repository you want to manage work for:

```text
/work-init --dry-run     # see exactly what it would do, writing nothing
/work-init               # do it
/work-ingest --dry-run   # backfill intake from a PRD, TODO file, code comments
```

If the project is one app inside a larger repository, start the session in the app's own
directory rather than at the repository root — see
[Monorepos and partial repositories](#monorepos-and-partial-repositories).

`/work-init` surveys the repo, interviews you about the things it cannot infer, writes
`docs/work/project.yml`, scaffolds the four work files, and generates `.claude/agents/*.md`
for your roster. It is built for repositories that already have work in them: it requires a
clean working tree so everything it writes is one reviewable diff, it never overwrites an
existing work file, it extracts project knowledge out of hand-written agent files before
regenerating them, and it treats two agents owning the same path as a hard stop. Those
safeguards are explained in [DESIGN.md](DESIGN.md#adopting-a-repository-that-already-has-work-in-it).

`/work-ingest` then backfills `requests.md` from whatever already holds the project's
outstanding work — a PRD, `TODO.md`, code comments, an old planning folder. Crucially it
**verifies every candidate against the codebase** before writing it, so a PRD describing a
product that half exists doesn't produce sixty requests for features that shipped months ago.
See [DESIGN.md](DESIGN.md#backfilling-existing-work) for how that pass works.

## Monorepos and partial repositories

A project managed by this plugin need not be a whole repository. It can be one app in a
monorepo, or one package in a workspace, with siblings that belong to other people and must
not be touched.

That case turns on separating two roots the model would otherwise treat as one:

| Root | What it is |
|---|---|
| **VCS root** | where `.git` lives. Branches, tags, pushes and merges are repository-wide by nature and happen here. |
| **project root** | where this project's world begins and ends. Every manifest path resolves against it, and nothing writes outside it. |

On an ordinary single-project repository they are the same directory and none of this
applies. `scope.root: null` is the default, and everything behaves as it always has.

### Setting it up

Run `/work-init` **from the project's own directory**, not the repository root:

```bash
cd ~/dev/toolA/internal-tools/apps/toolA
```

`/work-init` compares that directory with the VCS root, notices it is inside a larger
repository, and asks whether the repository holds other projects you must not touch. Answer
yes and it writes:

```yaml
scope:
  root: apps/toolA        # relative to the VCS root
  writes_outside: []      # nothing outside apps/toolA may be written
  agents_dir: project     # roster at apps/toolA/.claude/agents/
```

**Git does not need the session to start at the repository root.** It walks up to find `.git`
by itself, so working from `apps/toolA` gives you full commit, branch, tag and push while
keeping every file operation inside the app. Starting at the repository root to "get git
working" is the thing `scope` exists to make unnecessary — and starting there finds no
manifest anyway, since each member keeps its own.

### Every path is relative to the project root

`paths.work: docs/work` means `apps/toolA/docs/work`. `version.file: package.json` means
`apps/toolA/package.json`. An agent owning `src` owns `apps/toolA/src`.

**A leading slash anchors to the VCS root instead**, exactly as in `.gitignore` — for the few
things that genuinely live above the app:

```yaml
paths:
  changelog: CHANGELOG.md              # apps/toolA/CHANGELOG.md
release:
  pipeline:
    definition: /.github/workflows/release.yml   # shared, at the repo root
```

The point is that a member's manifest reads exactly like a standalone project's. Lift
`apps/toolA` out into its own repository and the only line that changes is `scope.root`.

### What the boundary actually does

- **Writes are fenced.** Nothing — no command, no spawned agent — writes outside the project
  root unless the exact path is listed in `scope.writes_outside`. Generated agent files carry
  the boundary as their first scope-table row.
- **Reads are not.** Building against a shared package means reading it, so an allowlist for
  reads would be pure friction.
- **`git status` is scoped.** A repo-wide check reports a sibling app's uncommitted work as
  your dirt, which turns a clean-tree precondition into a blocker nobody in the session can
  clear. Every check is scoped to the project root instead.
- **Sweeps are banned.** `git add -A`, `git add .` and `git commit -a` are forbidden outright
  when `scope.root` is set. Every path is named.
- **Surveys are scoped.** `/work-init` looks for version files inside the app rather than
  offering you thirty `package.json` files, and `/work-ingest` sweeps the app for `TODO.md`
  rather than ingesting another team's backlog under your IDs.
- **Agent names are prefixed** with the project slug — `toola-backend-developer` — so two
  members' rosters can never be confused for each other.

### Tags are repository-global — namespace them

This is the one thing that bites regardless of the boundary. A tag name belongs to the whole
repository, so the first time two apps both reach 1.2.3 the second cannot tag at all:

```yaml
version:
  tag_template: "toolA/v{version}"    # or "{slug}-{version}", or whatever the repo already does
```

`{version}` renders per `version.scheme`; `{slug}` is the project name lower-cased and
hyphenated. `/work-init` reads your existing `git tag` list first and offers whatever
convention the repository already follows.

**Check your pipeline when you set this.** A workflow triggered on `v*` will *not* fire on
`toolA/v1.2.3`, and a release that tags cleanly and builds nothing is the worst outcome here
— it looks shipped. `/work-init` and `/work-release` both read the workflow named in
`release.pipeline.definition` and cross-check its tag filter against the template before
anything is tagged.

### Working two apps at once

One checkout per app is a normal arrangement, and `git worktree` does it more cheaply than a
second clone — one object store, and worktrees structurally prevent two checkouts sharing a
branch:

```bash
git -C ~/dev/toolA/internal-tools worktree add ~/dev/toolB/internal-tools main
```

Either way, release branches already carry the project slug (`release/toola-1.2.3`), so two
members' releases never collide on a shared remote. What does need care is the base branch:
`/work-release` fetches before merging and reports whether the base had moved, because the
other checkout may have pushed since this one last looked.

### Upgrading an existing project

`scope` and `version.tag_template` arrived with `model_version: 4`; `vcs.stages` with `5`.
Existing manifests keep working: run `/work-init --upgrade`, and on an ordinary repository it
adds `scope.root: null` and `tag_template: null` — which preserve current behaviour exactly —
and expands `vcs.owner` plus `vcs.branching` into the six stages by table lookup, which needs
no decisions from you.

Where it finds a manifest that has been living *below* its repository root — a monorepo member
managed as though it were a whole repository — it says so, proposes the `scope.root`, and
offers to strip the now-redundant `apps/toolA/` prefix from every path. It shows you the full
before/after list and will not do it silently: a wrong root breaks every ownership table at
once.

## Release stages — who does what

A release performs six things, and `vcs.stages` says for each one **separately** whether it
happens and who does it. There is no global "who owns version control" switch, because that
question legitimately has six different answers:

```yaml
vcs:
  system: git
  stages:
    branch:       human    # you cut it
    commit:       agent    # the release coordinator commits
    push:         agent    # and pushes
    merge:        none     # no local merge
    pull_request: agent    # it opens the PR
    tag:          none     # nobody tags — the version lives in package.json
  delete_branch: after-merge
```

| Value | Meaning |
|---|---|
| `none` | it does not happen |
| `agent` | the release coordinator performs it |
| `human` | the session **stops**, tells you exactly what to run, waits for confirmation, **verifies it landed**, and continues |

`agent` names *who acts* — the AI rather than you. It never means a sub-agent gets spawned; the
release coordinator is a persona `/work-release` adopts.

**Stage order is `branch → commit → push → merge`/`pull_request` → `tag`.** Tag timing is
derived from that order rather than configured: because tagging follows integration, a
`human`-owned merge automatically defers the tag until the merge lands.

**Consecutive `human` stages become one handoff**, not three waits. Interleaving them — a
`human` push followed by an `agent` tag — is allowed but genuinely pauses the session
mid-release, and `/work-release` says so up front rather than surprising you at the point it
happens.

`/work-init` offers named shapes (trunk, release branch, pull request, "you cut branches and I
do the rest", "you own the repository") and writes the six fields explicitly. There is no preset
field in the schema — that would re-bundle exactly what the stage table exists to unbundle.

### Rules the manifest is checked against

- `commit: human` forces every later stage to `human` or `none` — nothing downstream works
  without commits, which makes this the total-handoff shape
- `commit: none` is invalid
- `merge` and `pull_request` may not both be active; either requires `branch`; `pull_request`
  also requires `push`
- `tag: none` requires `version.file`, or the version has nowhere to live
- any `human` stage makes `/work-crunch` refuse — an unattended loop cannot wait on you

**Tagging before a pull request merges is not supported and is not configurable.** The tag
would point at a commit that may never reach the base branch, or that a squash or rebase
rewrites out from under it.

### A branch with no integration stacks

`branch: agent` with both `merge` and `pull_request` set to `none` is a valid and useful shape
— cut a branch, commit, push, stop. But a release branch is cut from the current `HEAD`, and
nothing checks out the base afterwards, so **the next release branches from the last one.**
Across several releases you get a linear stack rather than independent branches, and merging
the newest brings the older ones with it.

`/work-release` reports the base it cut from as a resolved ref every time, and says explicitly
when that base is another release branch. `/work-crunch` warns before a multi-release run in
this configuration and offers to cap itself at one release.

## Commands

In the order you meet them: set up, backfill, then the delivery loop, then periodic
maintenance.

| Command | Does |
|---|---|
| `/work-init` | Sets a project up. `--dry-run` writes nothing; `--repair`/`--upgrade` (same flag) regenerates agents after a manifest change, upgrading a stale manifest first if the schema has moved. |
| `/work-ingest` | One-time backfill of `requests.md` from a PRD, TODO file, code comments, or a superseded planning folder. Verifies each candidate against the codebase first. |
| `/work-plan` | Closes out a finished release, checks blocked items, refines intake into work items, proposes the next release scope. |
| `/work-release` | Confirms scope, briefs and runs the required agents, verifies against acceptance criteria, then ships or hands off per the manifest. |
| `/work-spike` | Runs investigation-only items, producing one Findings/Recommendations document each. Refuses to run non-spike work. |
| `/work-crunch` | Loops plan → deliver → close out until the backlog empties or a guardrail stops it. Asks for its permissions once up front. Needs every release stage owned by `agent` or `none`, and a changelog. |
| `/work-review-deferred` | Re-triages parked items — keep, block, promote, or reject. The only thing that surfaces deferred work. |
| `/work-prune` | Trims completed items whose delivery is already in the changelog or tags. |
| `/work-migrate` | Updates a `project.yml` written against an older plugin version to the current schema, preserving its comments. Runs automatically inside `/work-init --upgrade`; call it directly to review the manifest edit on its own. |

## Skills

**`work-capture`** — ambient intake. Triggers on "log this", "note this down", "I spotted a
bug" and appends a correctly-formatted request without starting a session.

**`work-model`** — the model reference. Consult it for status meanings, field schemas, the
manifest schema, and conformance validation.

Commands are the deliberate ceremonies; the capture skill is the one thing you want firing
without being named.

## What the manifest controls

- project name, one-line description, product truth source, optional domain-rules file
- the project boundary — whether the project is the whole repository or one member of it,
  what may be written outside it, and where the agent roster lives
- paths for work, spikes, architecture, decisions, product docs, changelog
- ID prefixes and padding
- request and work item type vocabularies, priority vocabulary
- version control: whether the project is under it at all (`vcs.system: git | none`), and
  then, per release stage, whether it happens and who does it — branch, commit, push, merge,
  pull request, tag
- versioning, separately: scheme, who assigns the number, which file holds the version of
  record, which other files are kept in step with it, and the shape of the release tag
- release mechanics: changelog requirement and manual deploy steps
- the pipeline — as a path to its workflow definition where one exists, so `/work-release`
  reads what will actually happen rather than reciting a stale description
- a link to the project's own testing policy document, and whether a testing agent exists
- the agent roster with per-agent ownership, model, stack file, and consult triggers
- roles that deliberately do **not** exist, and where their work routes instead
- responsibilities reserved for the human

### Adding a project fact

It goes in the manifest. If a command or an agent file would have to change to record
something true of one project only, that is the drift this plugin exists to prevent.

## Design notes

[DESIGN.md](DESIGN.md) covers the decisions behind the model and why each one is the way it
is — among them:

- why scope-enforcement tables are derived from the ownership map rather than hand-written
- why roles that *don't* exist are declared explicitly
- why versioning and version control are separate concerns, and why the version is written at
  the start of a release rather than at the end
- why the schema gate refuses to auto-migrate a stale manifest
- why `/work-crunch` drives the other commands instead of reimplementing them, and the three
  rules it will not let you configure away

## Repository layout

```text
work-management/
  .claude-plugin/plugin.json
  commands/                 work-init, work-ingest, work-plan, work-release,
                            work-spike, work-crunch, work-review-deferred,
                            work-prune, work-migrate
  skills/
    work-capture/SKILL.md
    work-model/SKILL.md
      references/model.md              the portable model
      references/config-resolution.md  discovery + compatibility gate
      references/schema-history.md     every schema era and its migration
  templates/
    project.yml             the manifest, fully commented
    requests.md  backlog.yml  active-release.md  work-README.md
    agents/                 product-manager, principal-architect,
                            implementer, inactive-agent
    PLACEHOLDERS.md         how /work-init fills each template
```

## Releasing

Pushing a `.claude-plugin/plugin.json` version bump to `main` triggers
[`.github/workflows/release.yml`](.github/workflows/release.yml), which tags `v<version>` and
creates a GitHub Release from `CHANGELOG.md`. Do both edits in the same commit, in this order:

1. In `CHANGELOG.md`, rename `## [Unreleased]` to `## [<version>] - <date>` and add a fresh
   empty `## [Unreleased]` above it for whatever lands next.
2. Bump `version` in `.claude-plugin/plugin.json` to match.

If the version header and the `plugin.json` version don't match, the release notes silently
fall back to "See CHANGELOG.md." instead of the real entry — so bump both together, never one
without the other.

Full history is in [CHANGELOG.md](CHANGELOG.md).

## License

MIT. See [LICENSE](LICENSE).
