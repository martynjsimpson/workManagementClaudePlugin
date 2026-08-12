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
| `/work-crunch` | Loops plan → deliver → close out until the backlog empties or a guardrail stops it. Asks for its permissions once up front. Needs `vcs.owner: command` and a changelog. |
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
- paths for work, spikes, architecture, decisions, product docs, changelog
- ID prefixes and padding
- request and work item type vocabularies, priority vocabulary
- version control: whether the project is under it at all (`vcs.system: git | none`), who
  operates it, and the branching model
- versioning, separately: scheme, who assigns the number, which file holds the version of
  record, and which other files are kept in step with it
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
