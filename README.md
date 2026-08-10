# work-management

A project-agnostic `Request -> Work item -> Active release -> Done` delivery model for
AI-supported development.

The model is small on purpose: four files, three status planes, and enough structure that
agents can deliver from it without a planning-document sprawl. What makes it portable is that
**every project-specific fact lives in one manifest**. The commands, the skills, and the model
document contain no paths, no project names, no agent names, and no release mechanics.

## The three layers

| Layer | Where it lives | Changes when |
|---|---|---|
| Engine — commands, skills, templates | this plugin | you improve the model |
| Manifest — every project fact | `docs/work/project.yml` in the repo | you set up or change a project |
| Content — requests, backlog, release, agents | the repo | you do the work |

Update the plugin once and every project benefits. Adding a project fact never means editing
a command.

## Install and set up

Install the plugin, then in the repository:

```
/work-init --dry-run     # see exactly what it would do, writing nothing
/work-init               # do it
/work-ingest --dry-run   # backfill intake from a PRD, TODO file, code comments
```

It surveys the repo, interviews you about the things it cannot infer, writes
`docs/work/project.yml`, scaffolds the four work files, and generates `.claude/agents/*.md`
for your roster.

### Adopting into a repository that already has work in it

`/work-init` is built for this case, and it protects existing content in four ways:

- **It requires a clean working tree** (read-only `git status`), so everything it writes
  shows as one reviewable diff you can discard wholesale. Run `--dry-run` first if you'd
  rather see the output before it exists. On a project with no version control it takes
  timestamped backups of the files it will touch instead, and tells you where they are.
- **It never overwrites a work file.** Existing `requests.md`, `backlog.yml`, and
  `active-release.md` are left exactly as they are.
- **It extracts before it generates.** A hand-written agent file usually holds project
  knowledge the manifest has no field for — interface conventions, response envelopes,
  domain formulas, UI consistency rules. On a fresh adoption the `stack` and `domain_rules`
  files those belong in do not exist yet, so generating over the agent would destroy content
  whose new home hasn't been built. Instead it classifies every section, shows you the
  classification, writes the extracted content to its new home, verifies it landed, and only
  then generates. Decline the extraction and it leaves the file untouched.
- **Ownership overlap is a hard stop.** Two agents owning the same path produces two
  contradictory scope tables that each tell the other to keep out, which is painful to
  diagnose from the symptom. Legitimate carve-outs — a test role owning test files inside a
  developer's tree — are declared with `excludes` on both sides so a check can see them.

Then run `/work-ingest` to backfill `requests.md` from whatever already holds the project's
outstanding work.

### Backfilling existing work

`/work-ingest` is a two-pass operation, and the second pass is the point of it. It extracts
candidates from a PRD, `TODO.md`, code comments, or an old planning folder — then **verifies
each one against the codebase** before writing anything.

That matters because a PRD describes the *intended* product, and on an established project much
of it already exists. Ingesting naively produces sixty requests for features that shipped
months ago, which is worse than an empty file: it looks authoritative, and the first
`/work-plan` session starts refining work that is already done. Anything found to be built is
omitted and reported with its evidence, so the judgement is visible and challengeable.
Partially-built items are the highest-value output — the gap is the thing nobody had written
down.

Requests are ingested one per capability, with the individual requirements preserved as prose,
because splitting into work items is `/work-plan`'s job. Everything lands as `inbox`; nothing
is written straight to `backlog.yml`.

A source that exists *only* to capture work — `TODO.md`, `ROADMAP.md`, a superseded planning
folder — is moved to `.work-ingest-backup-<DATE>/` once its requests are on disk, so intake
lives in one place. Mixed-content sources are never moved: not the PRD, not a README, not a
code file with a `TODO` comment, and never anything the manifest points at.

## Commands

| Command | Does |
|---|---|
| `/work-ingest` | One-time backfill of `requests.md` from a PRD, TODO file, code comments, or a superseded planning folder. Verifies each candidate against the codebase first. |
| `/work-plan` | Closes out a finished release, checks blocked items, refines intake into work items, proposes the next release scope. |
| `/work-release` | Confirms scope, briefs and runs the required agents, verifies against acceptance criteria, then ships or hands off per the manifest. |
| `/work-spike` | Runs investigation-only items, producing one Findings/Recommendations document each. Refuses to run non-spike work. |
| `/work-review-deferred` | Re-triages parked items — keep, block, promote, or reject. The only thing that surfaces deferred work. |
| `/work-prune` | Trims completed items whose delivery is already in the changelog or tags. |
| `/work-init` | Sets a project up. `--dry-run` writes nothing; `--repair` regenerates agents after a manifest change. |
| `/work-migrate` | Updates a `project.yml` written against an older plugin version to the current schema, preserving its comments. |

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

Statuses are **not** configurable. They are the model.

## Design decisions worth knowing

**Scope-enforcement tables are derived, not written.** Each agent's redirect table is computed
by inverting the ownership map. Adding an agent means editing the manifest and running
`/work-init --repair` — not editing every existing agent file. Hand-maintained redirect tables
across N agents is an O(N²) consistency problem, and it is the first thing to rot.

**Absent roles are declared explicitly.** `inactive_agents` generates a tombstone file for
roles like `test-engineer` or `devops-engineer`. Without one, agents assume the role exists,
write briefs for it, and wait on a handoff that never comes. Declaring the absence is cheaper
than debugging the silence.

**The release coordinator is a persona, not an agent.** `/work-release` *is* the coordinator
for the duration of the session. It is never spawned as a sub-agent, and it is never listed in
`inactive_agents` either — listing it as absent would imply it could have been present.

**Versioning and version control are separate concerns.** An app can carry a version in
`package.json` and not be under git; a repo can be under git with the version expressed only as
a tag. Neither implies the other, so `vcs:` and `version:` are independent blocks and no command
infers one from the other.

**The version is written at the start of a release, not at the end.** `/work-release` confirms
the number immediately after you approve scope, then writes it to `version.file` and every path
in `version.mirrors` before any implementation begins — so everything built, run, or tested
during the session already carries the number it will ship as. Bumping at ship time means every
artefact produced during the release claims the previous version.

That has one consequence the command handles explicitly: an abandoned release has already
bumped the version, so restoring the previous value is part of the abandonment note rather than
optional tidying. Under `vcs.system: none` there is no diff to consult, so the previous value is
recorded in the Step 2 report and carried verbatim into the note.

**The schema is versioned, and the gate is real.** `model_version` is the schema contract. Every
command checks it before reading anything else and stops on a mismatch, pointing at
`/work-migrate`. Nothing auto-migrates — rewriting the manifest as a side effect of a planning
session is a surprise, and the manifest is the one file meant to stay human-authoritative.

A newer-than-plugin manifest is also a stop, not a shrug: a stale plugin silently ignoring a
field it does not recognise is worse than a refusal.

`/work-migrate` edits the file **textually rather than round-tripping the YAML**, because a
manifest in real use carries hand-written comments explaining why the project is configured as it
is — and a parse-and-redump destroys every one of them while producing a file that parses
identically. It asks about anything with no correct default (`version.mirrors` especially) rather
than filling it in.

**Version control is optional.** `vcs.system: none` is a first-class configuration, not a
degraded one. Every VCS step is then skipped rather than failed — no command asks for a
commit, a tag, or a clean tree, and `version.file` becomes required since there is no tag to
hold the number.

Two things change as a consequence, and the commands handle both. `/work-init` takes file
backups instead of relying on a clean tree, because there is no diff to discard. And the
changelog becomes the *only* narrative record of what shipped, which makes `/work-prune`
genuinely destructive — so it refuses to prune at all without a changelog, and retains more
by default.

**The manifest points at codebase facts rather than copying them.** What needs test coverage
belongs with the project's technical documentation, so `testing.policy_document` is a link and
the commands read it. Same for the pipeline: `release.pipeline.definition` is a path to the
real workflow file, which stays accurate as the workflow changes, where a prose description
would not. Anything the manifest can reference instead of restating, it references.

## Adding a project fact

It goes in the manifest. If a command or an agent file would have to change to record
something true of one project only, that is the drift this plugin exists to prevent.

## Layout

```
work-management/
  .claude-plugin/plugin.json
  commands/                 work-init, work-migrate, work-ingest, work-plan,
                            work-release, work-spike, work-review-deferred,
                            work-prune
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

## License

MIT, with the [Commons Clause](https://commonsclause.com/). Free to use, modify,
and self-host — including for your own commercial work — but you may not sell
the plugin itself or a product/service whose value comes substantially from it.
See [LICENSE](LICENSE).
