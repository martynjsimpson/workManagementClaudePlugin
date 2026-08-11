# Changelog

All notable changes to this plugin are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.0.0/). Versions correspond
to `version` in [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json) —
bumping that field is what ships a new version to installed users and, via
the release workflow, tags a GitHub Release.

## [Unreleased]

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
