# Changelog

All notable changes to this plugin are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.0.0/). Versions correspond
to `version` in [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json) —
bumping that field is what ships a new version to installed users and, via
the release workflow, tags a GitHub Release.

## [Unreleased]

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
