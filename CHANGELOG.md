# Changelog

All notable changes to this plugin are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.0.0/). Versions correspond
to `version` in [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json) —
bumping that field is what ships a new version to installed users and, via
the release workflow, tags a GitHub Release.

## [1.4.1] - 2026-08-18

### Fixed

- **Upgrading a `model_version: 3` manifest could not reach 5.** Era D's transition in
  `schema-history.md` listed only the moves to era E — `scope`, `version.tag_template` — and
  stopped. Its heading had been renamed to "To reach F" when era F was added, but the stage
  expansion underneath it never was. So `/work-init --upgrade` on a version-3 manifest would
  add the `scope` block, add `tag_template`, write `model_version: 5`, and leave `vcs.owner`
  and `vcs.branching` in place with no `vcs.stages` at all.

  It failed safely rather than silently: `/work-migrate` Step 5 verifies against the shape
  cross-check, which requires a `stages:` map, so the migration stopped and left the backup
  in place. But a failed migration needing diagnosis is not an upgrade path, and era D is the
  most-used one — it is what every project not yet on 1.3 is running.

  Eras C, B and A were unaffected; each already said "chain through D → E → F" explicitly.
  Only the era directly below the new one was wrong, which is precisely the entry whose
  heading rename looks sufficient and is not.

- **Two rules added for introducing a schema era**, because this failure recurs by
  construction. Adding an era now requires fixing the era immediately below it — which had no
  transition before, having been current — and walking every era bottom-up, following each
  chain literally rather than trusting its heading. All five chains were re-verified against
  the current schema.

## [1.4.0] - 2026-08-18

### Added

- **Per-stage release ownership.** `vcs.owner` asked one question — agent or human — and
  `vcs.branching` bundled *whether* each version-control action happens with *which* set of
  them happens at all. Between them they offered three shapes out of roughly fifteen sensible
  ones, and none of the obvious wants was expressible: cut the branch yourself while the agent
  commits and opens the pull request; push a release branch and stop; never tag at all because
  the version lives in `package.json` and the pipeline fires on a branch push.

  Enablement and ownership are one question asked twice, so `model_version: 5` replaces both
  fields with `vcs.stages` — six stages, each valued `none | agent | human`:

  ```yaml
  vcs:
    stages:
      branch: human        # you cut it
      commit: agent        # the release coordinator commits
      push: agent          # and pushes
      merge: none
      pull_request: agent  # it opens the PR
      tag: none            # nobody tags
    delete_branch: after-merge
  ```

  `none` = it does not happen. `agent` = the release coordinator performs it. `human` = the
  session stops, states exactly what to run, waits for confirmation, verifies it landed, and
  continues.

- **`vcs.delete_branch`**, so a merged release branch can survive. Previously forced.

### Changed

- **`command` is now `agent` throughout**, in `vcs.stages.*` and `version.owner` alike.
  `command` was internal jargon leaking into a user-facing value. The word names *who acts* —
  the AI rather than the human — and never authorises spawning a sub-agent: the release
  coordinator remains a persona `/work-release` adopts. `testing.agent` uses the word in its
  other sense, naming a roster member, and the two do not interact.
- **Tag timing is derived, not declared.** Because tagging follows integration in the stage
  order, a `human`-owned merge defers the tag until it lands — which is exactly what the old
  `branching: pr` mode did by special case. No timing field was needed.
- **`/work-release` walks the stages** instead of branching on three presets. It reports the
  whole stage plan in Step 2, before anything is written, and names any interleaved ownership
  — an `agent` stage between two `human` ones — as a session that will stall mid-release.
  Consecutive `human` stages collapse into one handoff rather than several waits.
- **Every human→agent boundary is verified, not trusted.** The branch gate already insisted on
  confirmation "not on an assumption, a 'will do', or silence"; that standard now applies at
  every boundary, with a per-stage verification table. An agent about to tag a merge it did not
  perform confirms the merge actually landed first.
- **A `human`-owned final stage completes the release in-session.** Previously the
  pull-request flow required a second run to move the status line, which left it at
  `ready-for-release` indefinitely whenever nobody came back — and that line is the release's
  external interface. An open pull request remains the one honest exception.
- `/work-crunch` refuses when **any** stage is `human`, naming the offending stages, rather
  than checking a single `vcs.owner`. A `human`-owned push or tag strands an unattended run
  just as completely as the branch gate did, and later in the cycle.
- `/work-init` settles stages by offering **named shapes** — trunk, release branch, pull
  request, "you cut branches and I do the rest", "you own the repository" — and writes the six
  fields explicitly. There is deliberately no preset field in the schema; that would re-bundle
  what the stage table exists to unbundle. Tagging is asked separately, because `tag: none` is
  an ordinary answer a preset should not quietly decide.
- Generated agent files resolve `{{VCS_CONSTRAINT}}` from `vcs.stages.commit` alone. It is the
  only stage an implementer's work depends on; enumerating the other five is noise.
- **Release branches that never integrate now say so.** `branch: agent` with no `merge` or
  `pull_request` is a valid shape — cut, commit, push, stop — but a release branch is cut from
  the current `HEAD` and nothing checks out the base afterwards, so the next release branches
  from the last one. Across several releases that is a linear stack rather than independent
  branches, and merging the newest brings the older ones along. It is invisible in a single
  manual release and easy to inherit unknowingly from a loop.

  `/work-release` now reports the base as a resolved ref — `cut from main @ a1b2c3d` — rather
  than as "the current branch", because `HEAD` has moved by the time anyone reads the report.
  Where the base is itself a release branch it says so explicitly and names the consequence.
  `/work-crunch` warns before a multi-release run in this configuration and offers to cap the
  budget at one, taking the answer either way rather than capping itself.

### Unchanged by design

Tagging before a pull request merges remains unsupported and is not configurable. The tag would
point at a commit that may never reach the base branch, or that a squash or rebase rewrites out
from under it.

### Migration

`/work-init --upgrade` expands the two old fields into the six new ones. It is a mechanical
table lookup — all six combinations of `vcs.owner` and `vcs.branching` have exactly one
destination each, and nothing needs a human decision. The full table is in
`skills/work-model/references/schema-history.md`.

`/work-migrate` shows the expansion as a table rather than a list of renames, since one answer
becoming six is not otherwise legible, and offers — once, without pressing — to change any
stage now that per-stage ownership is expressible for the first time.

## [1.3.0] - 2026-08-18

### Added

- **Monorepo support, via a project boundary that is separate from the repository.** The model
  assumed a project *was* a repository, and every path resolved against the repository root.
  On a monorepo that assumption failed in six places at once: a clean-tree check reported a
  sibling app's uncommitted work as this project's dirt, `/work-init` surveyed every member's
  version file, every ownership path had to carry an `apps/toolA/` prefix, `/work-ingest` swept
  another team's `TODO.md` into these IDs, `/work-prune` read a tag history belonging to
  several projects, and `v1.2.3` was a name the whole repository shared. Those were one
  problem — two roots treated as one — so `model_version: 4` adds a `scope:` block naming the
  second:

  - `scope.root` is the project root relative to the VCS root, or null for an ordinary
    single-project repository, which is the default and preserves existing behaviour exactly.
  - **Every path field now resolves against the project root**, with a leading slash anchoring
    to the VCS root as it does in `.gitignore`. A member's manifest therefore reads
    byte-identically to the standalone version of itself; lifting the app into its own
    repository changes one line.
  - `scope.writes_outside` lists the paths outside the boundary this project may write —
    empty by default. Writes are fenced, reads are not: building against a shared package
    means reading it, while writing to a sibling app being edited in another checkout is the
    actual hazard.
  - `scope.agents_dir` chooses between `<scope.root>/.claude/agents/` and the repository root,
    and agent names gain the project slug as a prefix so two members' rosters cannot be
    confused for each other.

- **`version.tag_template`, because a tag name is repository-global.** The first time two
  members both reach 1.2.3, the second cannot tag at all. The field takes `{version}` and
  `{slug}` — `"toolA/v{version}"` — and defaults to null, which renders the bare version as
  before. `/work-init` reads the existing `git tag` list and offers whatever convention the
  repository already follows rather than inventing one.

  Both `/work-init` and `/work-release` **cross-check the rendered tag against the pipeline's
  trigger filter before anything is tagged**. A workflow watching `v*` does not fire on
  `toolA/v1.2.3`, and a release that tags cleanly, builds nothing and looks shipped is the
  failure here that nobody notices for days.

### Changed

- Every git check is now scoped to the project root — `git status --porcelain -- <root>` —
  across `/work-init`, `/work-release`, `/work-ingest` and `/work-migrate`. `git add -A`,
  `git add .`, `git add :/` and `git commit -a` are forbidden outright where `scope.root` is
  set; every staged path is named and confirmed inside the boundary first.
- Version control is now detected with `git rev-parse`, not by looking for a `.git` directory
  beside the manifest. The old test reported "no version control" on a monorepo member, whose
  repository legitimately sits several levels above it.
- Discovery reports sibling projects instead of an unmanaged repository. Walking up from a
  monorepo root correctly finds no manifest while several sit two levels down, so
  `config-resolution.md` now scans downward before declaring the repository unmanaged, and
  names what it found rather than picking one. Selecting a member on the human's behalf is how
  work lands in the wrong project; the working directory is the unambiguous statement of
  intent, which is why there is no `--project` flag.
- `/work-release` fetches the base branch before merging into it and reports whether it had
  moved. Where the same repository is checked out twice — the usual arrangement when two of
  its apps are worked in parallel — merging onto a stale ref silently builds a release on a
  tree nobody has.
- `/work-prune` filters the tag history by `version.tag_template`. Reading it unfiltered gives
  a version history belonging to several projects, and a cutoff drawn from that prunes the
  wrong things in both directions.
- `/work-crunch` states the boundary back as part of its pre-flight facts, warns when an
  unattended run would claim repository-global tags, and may never widen
  `scope.writes_outside` to make a cycle proceed.
- Generated agent files carry the boundary as a paragraph before their owned-paths list and as
  the first row of the scope table — one catch-all row rather than an enumeration of sibling
  directories, which would go stale the day another is added. Owned paths are rendered
  resolved rather than in the manifest's project-relative shorthand: the manifest is written
  to stay portable, a prompt is written to be acted on.

### Migration

`/work-init --upgrade` brings a `model_version: 3` manifest to 4. On an ordinary repository
the change is purely additive — `scope.root: null`, `writes_outside: []`,
`agents_dir: project`, `tag_template: null` — and no path is rewritten.

Where the manifest is found *below* its repository root, it is a monorepo member that has been
managed as a whole repository. `/work-migrate` says so, proposes the `scope.root`, and offers
to strip the now-redundant prefix from every path, showing the full before/after list first. It
will not do it silently: a wrong root breaks every ownership table at once.

## [1.2.2] - 2026-08-13

### Fixed

- The work files were never committed by a release. `/work-release` committed code and
  version files and left `active-release.md`, `backlog.yml` and `requests.md` in the working
  tree — including the planning state `/work-plan` had written before the session, which the
  release branch simply carried across uncommitted. So a status line could move correctly on
  disk and still never move for anything reading the repository, and an interrupted release
  left its own state in a working tree rather than on the branch. `/work-release` Step 2b now
  commits those three files immediately after the branch gate and before the version bump, so
  the release's starting position is its own commit and the bump reads as a bump; every status
  transition after it is committed as it happens rather than batched into a tidy-up at ship
  time. `/work-spike` does the same for its transitions alongside the spike documents. Both
  name the paths explicitly rather than sweeping the tree, so a monorepo's unrelated changes
  stay put; both do nothing where `<vcs.system>` is `none`; and where `<vcs.owner>` is `human`
  the work files are now named in the handoff, which previously listed only code, version
  files and the changelog.

- The version bump was not committed when it was made. Step 2b wrote `<version.file>` and its
  mirrors before any agent was spawned — deliberately, so everything built in the session
  carries the shipping number — but the commit came at ship time, by which point the bump was
  mixed into whatever the implementers had produced. It now commits immediately and on its
  own, before the first agent is spawned, making it the first release-owned commit on the
  branch: the anchor a rollback rewinds to, the commit a bisect lands on, and a single
  `git revert` for an abandoned release rather than a hand-restore of every versioned path
  from a value buried in a session report. The abandonment note now records that commit's
  SHA, Step 7 confirms it is on the branch being tagged rather than re-committing the files,
  and `/work-crunch` names it when it stops a failed cycle.

## [1.2.1] - 2026-08-12

### Fixed

- A spike release left `active-release.md` sitting at `Status: approved` for the whole run.
  `/work-spike` wrote the status exactly once, as a prose line at the top of its wrap-up
  step, where `/work-release` writes it three times as its own instruction — so the release
  had no `in-progress` state at all, and its single transition to `ready-for-release` was
  the most droppable line in the command. Beyond leaving nothing outside the session able
  to tell a running spike from an idle one, the failure mode was quiet and compounding:
  `/work-plan`'s closeout keys off the top-level status alone, so at `approved` it skipped
  closeout entirely and refined-and-proposed straight over the release, leaving the spike's
  work items un-`done` in `backlog.yml` and their requests without their `SPIKE: <ITEM-ID>`
  completion value. `/work-crunch` reads `approved` as "run the delivery step", so an
  interrupted spike release was also a candidate to be run again from the top —
  `/work-release` gets that protection from its `in-progress` write, and spikes had no
  equivalent.

  `/work-spike` now sets `in-progress` before the first spike is assigned and
  `ready-for-release` as the first action of its wrap-up, both as constraints rather than
  passing mentions, giving `approved → in-progress → ready-for-release` against a code
  release's `approved → in-progress → testing → ready-for-release`. `/work-crunch` verifies
  the status actually reads `ready-for-release` before closing out, rather than asserting
  both delivery paths end there, and stops if a delivery step left it behind. The
  `work-model` skill and the `active-release.md` template now state that the top-level
  status is the release's external interface and that per-item statuses are not a
  substitute for it.

- `Status: approved` had no defined writer. `/work-plan` writes `proposed`, `/work-release`
  requires approval to exist but cannot record it — its branch gate bars writing
  `active-release.md` before the release branch exists — and `/work-crunch` said to "record
  the approval" without saying where. The value was reached by inference, which is fragile
  for a state `/work-crunch`'s entry table resumes from. `/work-plan` now writes it when the
  human approves in that session, `/work-crunch` writes it when its contract approves,
  `/work-release` is explicitly barred from it, and the model documents which command owns
  the transition.

## [1.2.0] - 2026-08-12

### Changed

- `/work-plan` no longer turns a release's "Out of scope" section into a second backlog. The
  section was defined in one line — "what a reasonable reader might assume is included but is
  not" — with no admission test and nowhere else for the PM to put its reasoning about the
  candidates it passed over, so in practice it filled up with every unselected `ready` item and
  its rationale. That duplicated `backlog.yml`, went stale as soon as it was written, and was
  read on every cycle by `/work-release`, `/work-spike` and `/work-crunch` as part of the
  implementer briefing. The definition now carries an explicit admission test, a prohibition on
  listing unselected backlog items, and a size expectation ("None." is a correct answer), in
  both the command and the `active-release.md` template. The reasoning that was landing there
  now has a home: Step 5's proposal report covers what else was ready and why it was not
  chosen, in the chat rather than in the file.

- Documentation only, no behaviour change. The README now opens with the model itself — the
  flow, the four work files, and the three status planes — before install and setup, and the
  command table is ordered as you meet the commands (set up, backfill, the delivery loop,
  then maintenance) rather than arbitrarily. The design rationale that made up roughly a
  third of the README moved to a new [DESIGN.md](DESIGN.md), linked from the sections that
  depend on it, so the README stays a reference and the reasoning stays available in full.

## [1.1.0] - 2026-08-11

### Added

- `/work-crunch` — runs plan → deliver → close out on a loop, so a backlog can be worked
  through in one sitting instead of one command at a time. It drives the existing commands
  rather than reimplementing them: each cycle reads `work-plan.md` and then either
  `work-release.md` or `work-spike.md` and follows it as written, so there is no second copy
  of the release logic to drift. Both delivery paths end at `ready-for-release`, so closeout
  is identical either way.

  Permissions are asked once up front — release budget, scope auto-approval, release size,
  version bump rule, manual-verification pause, spike confirmation, and a run-scoped
  `version.owner` override where the manifest reserves numbering for the human. Those answers
  are scoped to the run and never written to `project.yml`.

  It refuses to run where an unattended loop cannot be safe: `vcs.owner: human` stops at
  `/work-release`'s branch gate by design, and a run without a required changelog leaves no
  durable record of what it shipped. Under `vcs.branching: pr` it caps itself at one release,
  since the merge is the human's to time. A null `testing.policy_document` combined with no
  manual verification is a hard warning requiring explicit confirmation.

  Stop conditions are checked after every cycle: budget reached, backlog exhausted,
  verification failed, no measurable progress (the shippable work-item ID set unchanged, or a
  proposal repeating the previous cycle's items), a major version bump, or an architect
  trigger with no architect on the project. On failure it stops and leaves the release exactly
  as it stands — it never marks a release `abandoned`, reverts code, or restores a version.
  It never guesses a product decision either; a question with no human to answer it stays
  parked as a `needs-refinement` request and is surfaced in the final report.

  Spike work is a first-class path through the loop, not an exclusion. Because `/work-release`
  refuses a release containing spikes and `/work-spike` refuses one containing anything else,
  a mixed proposal would deadlock the loop from both sides — so planning selects all-spike or
  all-code scope for a given cycle, never both.

  The run is resumable: an interrupted crunch is re-entered by running the command again,
  which resolves its entry point from `active-release.md`. An `in-progress` or `testing`
  release found on entry stops and asks rather than resuming blind, since which agents
  completed is unknowable from the file.

### Fixed

- Completion of a spike had no defined value for `Done in:` / `done_in`. The field is
  specified as a release version, and a spike bumps none — so agents improvised prose like
  `spike completed 2026-08-03 (no release/code change)` into a field every other item parses
  as a version. Spike completion is now recorded as `SPIKE: <ITEM-ID>`: a fixed, comparable
  form that points at the work item whose document is the actual deliverable. The ID is
  stored rather than the path, since `<paths.spikes>/<ITEM-ID>.md` is derivable and a stored
  path goes stale the first time that setting moves. A `done_in` may hold both forms — a
  request satisfied by an investigation and then an implementation carries a spike marker
  alongside a version.

  This also fixes a false positive in `/work-prune`. Its Step 3 verifies each candidate
  appears in the changelog before removing it, so every spike-completed item was reported as
  a changelog gap, sending the human to add an entry that must not exist. Prune now checks
  the right durable record for a `SPIKE:` marker — that the spike document exists with both
  required sections — and never age-gates a marker against the version cutoff, since it has
  no version to compare. Legacy free-text values are listed for correction rather than
  interpreted; `/work-plan` rewrites them to the current form as it closes items out.

## [1.0.1] - 2026-08-11

### Fixed

- `/work-plan`: `blocked` requests were checked every planning session, but `partially-done`
  requests had no equivalent — once a request's remaining scope wasn't turned into a new
  request or work item on the spot, nothing ever came back to it, so it just accumulated
  silently. Added a dedicated step that checks every `partially-done` request each session,
  drafts a fresh inbox request for remaining scope that isn't already tracked, and corrects
  requests misfiled outside `## Refined requests`. Documented the same in `model.md` and
  `templates/requests.md`.

## [1.0.0] - 2026-08-10

First stable release. The model (schema `model_version: 3`) and command surface are
considered settled — this and future 1.x releases are docs, templates, and tooling
polish rather than behavioral changes, unless a changelog entry says otherwise.

### Added

- `SECURITY.md` — how to report a vulnerability privately, and the supported-versions
  policy (latest release only, via auto-update or `/plugin update`).

### Fixed

- `/work-init`: the "survey the repository" instruction got stranded under the `Step 1a`
  heading added in 0.9.0, positioned *after* Step 1a's own conclusion ("continue directly
  into Step 5"). An agent upgrading a stale manifest would hit a dangling, contradictory
  survey instruction on the way out. That survey only ever applies to the no-manifest path
  that reaches Step 2's interview, so it's now Step 2's lead-in instead, where it's
  unambiguous.
- `work-capture` skill: a broken line wrap left "malformed," orphaned on its own line
  mid-sentence, also from 0.9.0. Rejoined into one sentence.

### Changed

- Trimmed `templates/active-release.md`'s HTML-comment block by about a third. It's hidden
  from rendered Markdown, but still re-read by `/work-plan` and `/work-release` on every
  load, and most of its version/branch-ordering detail duplicated `/work-release`'s own
  Steps 2 through 2b almost verbatim. Kept the section skeleton in full — it's the actual
  scaffold `/work-plan` writes from — and compressed the status legend and version/branch
  paragraphs to pointers at the authoritative source. Same pass as `project.yml`,
  `backlog.yml`, and `requests.md` in 0.9.1/0.9.2.

Pre-v1.0 audit pass: read every command, skill, and template end to end looking for stale
cross-references, leftover verbosity, and formatting glitches. No field, status, or behavior
changed beyond the Step 1a fix above — doesn't require a schema bump.

## [0.9.3] - 2026-08-10

### Fixed

- Release workflow: bumped `actions/checkout` from `v4` to `v7`, clearing the "Node.js 20
  is deprecated" warning. Node 24 support landed in `actions/checkout@v5.0.0`; `v4` still
  ships a Node 20 action, so it was being force-run on a runtime it wasn't built for. No
  behavior change, just a clean run.

## [0.9.2] - 2026-08-10

### Changed

- Trimmed `templates/backlog.yml`'s field-reference comment from a full per-field
  description list (duplicating `model.md`'s annotated schema almost verbatim) to a
  compact field list pointing at the `work-model` skill for semantics. Kept the
  `blocked`/`deferred`/`done` rules in full since commands actually enforce them.
- Reformatted `templates/requests.md`'s status legend from prose bullets to a table,
  matching how `model.md` itself presents the same statuses, and tightened the intro.
  The legend stays complete — the model requires it at the top of every `requests.md` —
  just more compact.

Both files are live working files, not one-time scaffolding, so their comments get
re-read on every load; this is the same "say what to put where, point elsewhere for the
why" pass as `project.yml` in 0.9.1. No field, placeholder, or status changed — verified
`backlog.yml` still parses to the same data and every `{{PLACEHOLDER}}` is intact. Doesn't
require a schema bump.

## [0.9.1] - 2026-08-10

### Changed

- Trimmed the comments in `templates/project.yml` by about a third. The `vcs`,
  `version`, and `release` blocks had grown into essays duplicating rationale
  that already lives in the README and the `work-model` skill; this file
  lives in every adopting project, so it now says what to put where and
  points elsewhere for the why. `model_version`'s comment no longer carries
  plugin-maintainer instructions (schema-history.md already has those).
  No field, default, or structure changed — this does not require a schema
  bump, and existing manifests are unaffected.

## [0.9.0] - 2026-08-10

### Added

- `/work-init --upgrade`, an alias for `--repair` with identical behavior.

### Changed

- `/work-init --repair`/`--upgrade` now runs the schema compatibility gate before doing
  anything, instead of skipping straight to agent regeneration. A stale manifest no longer
  risks a silent misread from a moved key — it's the one condition that previously slipped
  past the gate every other command enforces.
- When it finds a stale manifest, `/work-init --repair`/`--upgrade` now performs the
  `/work-migrate` procedure itself, in the same session, before regenerating agents — so
  upgrading after a plugin version bump is one command instead of two you have to know to run
  in order.
- `config-resolution.md`'s schema-gate messages, and every other pointer to a stale schema
  across the commands and skills, now recommend `/work-init --upgrade` instead of bare
  `/work-migrate`.

`/work-migrate` is unchanged and still directly callable — for reviewing or controlling the
manifest edit on its own, separate from agent regeneration.

## [0.8.2] - 2026-08-10

### Changed

- Simplified the license from MIT + Commons Clause to plain MIT.

## [0.8.1] - 2026-08-10

### Added

- Release automation: a workflow tags a GitHub Release whenever `version`
  bumps in `.claude-plugin/plugin.json`, using the matching section of this
  file as release notes.

## [0.8.0] - 2026-08-10

### Added

- Initial public release, commands: `/work-init`, `/work-ingest`,
  `/work-migrate`, `/work-plan`, `/work-prune`, `/work-release`,
  `/work-review-deferred`, `/work-spike`.
- `work-model` and `work-capture` skills.
- Templates for `project.yml`, `requests.md`, `backlog.yml`,
  `active-release.md`, and the `.claude/agents/*.md` roster.
- Repo restructured to also serve as a self-hosted plugin marketplace
  (`.claude-plugin/marketplace.json`), so it installs via
  `/plugin marketplace add` instead of distributing the `.plugin` file
  by hand.
- MIT + Commons Clause license.
