# Work management — {{PROJECT_NAME}}

This directory runs the `Request -> Work item -> Active release -> Done` model.

| File | Purpose | Edited by |
|---|---|---|
| `project.yml` | The manifest. Every project-specific fact lives here and nowhere else. | Human, via `/work-init` |
| `requests.md` | Human-friendly intake. Rough is fine. | Human, or the `work-capture` skill |
| `backlog.yml` | Refined, agent-readable work items. | `/work-plan` |
| `active-release.md` | The single current delivery scope. | `/work-plan`, `/work-release` |

Spike documents live in `{{SPIKES_PATH}}`, one per spike work item.

The model itself is documented in the `work-management` plugin, not here. This file is
the local operating guide; the model is portable and identical across projects.

## Commands

| Command | Does |
|---|---|
| `/work-ingest` | Backfills intake from existing material — a PRD, TODO file, code comments, an old planning folder. |
| `/work-plan` | Closes out a finished release, checks blocked items, refines intake, proposes the next scope. |
| `/work-release` | Confirms scope, briefs and runs the agents, verifies, then ships or hands off. |
| `/work-spike` | Runs investigation-only items, producing one Findings/Recommendations document each. |
| `/work-crunch` | Loops plan → deliver → close out until the backlog empties or a guardrail stops it. Asks for its permissions once up front. |
| `/work-review-deferred` | Re-triages parked items — the only thing that surfaces them. |
| `/work-prune` | Trims completed items whose delivery is already durably recorded. |
| `/work-init --repair` / `--upgrade` | Regenerates the agent roster after a manifest change, upgrading a stale manifest first if the plugin's schema has moved. |
| `/work-migrate` | Updates this manifest to the current schema on its own, without touching agents. Runs automatically inside `/work-init --upgrade`. |

The `work-capture` skill handles quick intake without starting a session.

## If you are the human

Add requests freely. Do not worry about types, priorities, or work item status when
capturing — `/work-plan` sorts that out. Approve or change proposed release scope before
implementation starts. Avoid editing `backlog.yml` unless you mean to refine work directly.

Versioning is set in `project.yml` under `version:`, version-control behaviour under `vcs:`, and
changelog and deploy steps under `release:`. These are three independent concerns — change them
there, not in a command or an agent file.

The version is assigned and written at the *start* of a release, right after you approve scope,
so builds made during the session carry the right number.

## If you are an agent

Start from `backlog.yml` and `active-release.md`. Do not read superseded planning documents
by default. Do not create planning files, phase files, or side-car backlogs.

Your ownership boundaries are generated into your agent file from `project.yml`. When a task
falls outside them, redirect rather than reaching across — the boundaries exist so two agents
never edit the same path.

## Adding a project fact

It goes in `project.yml`. If you find yourself editing a command or an agent file to record
something true of this project only, stop: that is the drift this model is built to prevent.
Update the manifest and run `/work-init --repair`.
