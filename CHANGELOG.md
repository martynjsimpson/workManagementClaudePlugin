# Changelog

All notable changes to this plugin are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.0.0/). Versions correspond
to `version` in [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json) —
bumping that field is what ships a new version to installed users and, via
the release workflow, tags a GitHub Release.

## [Unreleased]

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
