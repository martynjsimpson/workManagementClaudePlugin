# Design notes

Why the model is shaped the way it is. The [README](README.md) covers what the plugin does;
this covers the decisions behind it, and the failure modes each one is there to prevent.

## Adopting a repository that already has work in it

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

## Backfilling existing work

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

## Scope-enforcement tables are derived, not written

Each agent's redirect table is computed by inverting the ownership map. Adding an agent means
editing the manifest and running `/work-init --repair` — not editing every existing agent
file. Hand-maintained redirect tables across N agents is an O(N²) consistency problem, and it
is the first thing to rot.

## Absent roles are declared explicitly

`inactive_agents` generates a tombstone file for roles like `test-engineer` or
`devops-engineer`. Without one, agents assume the role exists, write briefs for it, and wait
on a handoff that never comes. Declaring the absence is cheaper than debugging the silence.

## The release coordinator is a persona, not an agent

`/work-release` *is* the coordinator for the duration of the session. It is never spawned as a
sub-agent, and it is never listed in `inactive_agents` either — listing it as absent would
imply it could have been present.

## Versioning and version control are separate concerns

An app can carry a version in `package.json` and not be under git; a repo can be under git
with the version expressed only as a tag. Neither implies the other, so `vcs:` and `version:`
are independent blocks and no command infers one from the other.

## The version is written at the start of a release, not at the end

`/work-release` confirms the number immediately after you approve scope, then writes it to
`version.file` and every path in `version.mirrors` before any implementation begins — so
everything built, run, or tested during the session already carries the number it will ship
as. Bumping at ship time means every artefact produced during the release claims the previous
version.

That has one consequence the command handles explicitly: an abandoned release has already
bumped the version, so restoring the previous value is part of the abandonment note rather than
optional tidying. Under `vcs.system: none` there is no diff to consult, so the previous value is
recorded in the Step 2 report and carried verbatim into the note.

## `/work-crunch` drives the other commands, it does not reimplement them

It reads `work-plan.md`, `work-release.md` and `work-spike.md` and follows them as written, so
there is one copy of the release logic rather than two that drift. What it adds is a
permissions contract asked once up front, and stop conditions checked after every cycle — a
budget, an exhausted backlog, a failed verification, or no measurable progress.

Three of its rules are not configurable. It never guesses a product decision to keep the loop
moving — a question with no human to answer it stays parked as a `needs-refinement` request,
because a backlog of invented intent is worse than a short run. It never auto-bumps a major
version, since that is a promise to the project's users rather than an inference from a
`type:` field. And it never abandons a release: on a failed verification it stops and leaves
everything exactly as it stands, because abandonment means reverting code *and* restoring the
version, and an unattended agent undoing a working tree is the one action worth ruling out
entirely.

It requires `vcs.owner: command` — the branch gate is the human owning the repository, and no
run-scoped override can borrow that — and a required changelog, since that is the only durable
narrative of what a multi-release run shipped. Under `vcs.branching: pr` it caps itself at one
release, because the merge is yours to time.

## The schema is versioned, and the gate is real

`model_version` is the schema contract. Every command checks it before reading anything else
and stops on a mismatch, pointing at `/work-init --upgrade`. Nothing auto-migrates — rewriting
the manifest as a side effect of a planning session is a surprise, and the manifest is the one
file meant to stay human-authoritative.

A newer-than-plugin manifest is also a stop, not a shrug: a stale plugin silently ignoring a
field it does not recognise is worse than a refusal.

## `/work-init --repair` and `--upgrade` are the same flag, and both check the gate too

Regenerating agents off a manifest the gate would otherwise refuse is the same silent-misread
risk under a friendlier name, so `--repair`/`--upgrade` runs the gate first like everything
else. When it finds a stale manifest it doesn't just point elsewhere — it performs the
`/work-migrate` procedure itself, in the same session, then continues into agent regeneration.
That procedure edits the file **textually rather than round-tripping the YAML**, because a
manifest in real use carries hand-written comments explaining why the project is configured as
it is — and a parse-and-redump destroys every one of them while producing a file that parses
identically. It asks about anything with no correct default (`version.mirrors` especially)
rather than filling it in. Run `/work-migrate` directly instead when you want that edit
reviewed on its own, before anything touches the agent files.

## The project root and the VCS root are different things

The model originally assumed a project *was* a repository, and every path was relative to the
repository root. That holds right up until the project is one app in a monorepo, at which
point the assumption fails everywhere at once: a clean-tree check reports a sibling app's
uncommitted work as your dirt, `/work-init` offers thirty `package.json` files to pick a
version from, every ownership path carries an `apps/toolA/` prefix, and `v1.2.3` is a name
the whole repository shares.

Those look like six problems. They are one — two roots being treated as one — so `scope.root`
names the second root and the rest follows from it. When it is null the two coincide and
nothing changes, which is why existing manifests needed no path rewriting to reach schema 4.

**Paths resolve against the project root, not the repository root**, with a leading slash
escaping upward as it does in `.gitignore`. The alternative — keeping paths
repository-relative and letting each carry the prefix — was rejected because it makes the
manifest non-portable for no gain: under the chosen rule a member's manifest is byte-identical
to the standalone version of itself, and lifting the app into its own repository changes one
line.

**Writes are fenced; reads are not.** An allowlist for reads would be constant friction —
building against a shared package means reading it — while contributing nothing, since reading
a sibling app harms nobody. Writing to one is the actual hazard, particularly when that
sibling is checked out somewhere else and being edited right now.

**Blacklisting was the obvious alternative and is the wrong shape.** A denylist of sibling
apps is open-ended: it has to enumerate every peer, and it silently stops being correct the
day someone adds another. An allowlist anchored on one root is closed and stays correct
without maintenance — the same reasoning that makes agent ownership `owns` plus `excludes`
rather than a global ignore list.

**A repository-global namespace needs project-local names.** Tags, branches and commits all
live in a namespace shared with every other member, which is why release branches were already
slugged and why `version.tag_template` now exists. The trap worth naming: namespacing a tag can
silently disable the release pipeline, because a workflow watching `v*` does not fire on
`toolA/v1.2.3`. That produces a release that tags cleanly, builds nothing, and looks shipped —
so both `/work-init` and `/work-release` read the workflow file and cross-check its filter
against the template before anything is tagged.

## Version control is optional

`vcs.system: none` is a first-class configuration, not a degraded one. Every VCS step is then
skipped rather than failed — no command asks for a commit, a tag, or a clean tree, and
`version.file` becomes required since there is no tag to hold the number.

Two things change as a consequence, and the commands handle both. `/work-init` takes file
backups instead of relying on a clean tree, because there is no diff to discard. And the
changelog becomes the *only* narrative record of what shipped, which makes `/work-prune`
genuinely destructive — so it refuses to prune at all without a changelog, and retains more
by default.

## The manifest points at codebase facts rather than copying them

What needs test coverage belongs with the project's technical documentation, so
`testing.policy_document` is a link and the commands read it. Same for the pipeline:
`release.pipeline.definition` is a path to the real workflow file, which stays accurate as the
workflow changes, where a prose description would not. Anything the manifest can reference
instead of restating, it references.
