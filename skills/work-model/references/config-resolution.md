# Configuration resolution protocol

Every command and skill in this plugin begins by resolving the manifest. No command may
hardcode a path, project name, agent name, ID prefix, or release rule.

## Discovery order

Walk up from the current working directory to the repository root. At each level, test
in order:

1. `docs/work/project.yml`
2. `.claude/work/project.yml`
3. any `project.yml` sitting beside both a `requests.md` and a `backlog.yml`

Take the first match. Stop at the repository root (the directory containing `.git`).

In Cowork, the connected folder is the repository root — start there.

## When no manifest is found

Stop. Report exactly:

> No work-management manifest found. Run `/work-init` to set this project up.

Do not guess paths. Do not create files. Do not fall back to conventional locations. A
missing manifest is the one condition under which every command except `/work-init`
refuses to act — silently inventing a location is how the drift this plugin exists to
prevent gets reintroduced.

## Schema compatibility gate

**The supported schema is `model_version: 3`.** Check this before reading any other field and
before taking any action. A command that proceeds against a schema it does not understand will
misread a key that has moved — silently, and with a plausible-looking result.

| Condition | Action |
|---|---|
| `model_version` == 3 | Continue to the shape cross-check below. |
| `model_version` < 3 | **Stop.** Report: "This manifest uses schema version N; this plugin needs version 3. Run `/work-migrate` to update it." |
| `model_version` > 3 | **Stop.** Report that the manifest is newer than the plugin, and that the plugin should be updated. Do not proceed by ignoring unknown fields. |
| `model_version` absent | **Stop.** Treat as pre-3 and direct to `/work-migrate`. |

**Never auto-migrate.** Rewriting the manifest as a side effect of running `/work-plan` is a
surprise nobody asked for, and the manifest is the one file meant to stay human-authoritative.

**Shape cross-check.** A manifest can declare `3` and still be wrong — hand-edited, or
part-migrated. Confirm the current markers are present: a top-level `version:` block; `vcs:`
with `system`; `release:` carrying only `changelog`, `pipeline`, `deploy_steps`; `testing:` with
`policy_document`; `excludes` on every agent. If the declaration and the shape disagree, say so
and stop. Do not repair it inline.

For what each earlier era looks like and how it maps forward, read `schema-history.md`. Note
especially that `model_version: 2` covers four different shapes, because the number was left
unchanged across three breaking changes — so a version-2 manifest must be identified by its
keys, not by its declaration.

## Required fields

If a required field is missing (`project.name`, `project.description`,
`project.primary_reference`, `paths.work`, `paths.spikes`, `ids.*`, `taxonomy.*`,
`vcs.system`, `version.scheme`, `version.owner`, `release.*`), name the missing field and stop.
Do not substitute a default for a required field.

## The one exception: degraded mode for capture

The `work-capture` skill may proceed on a stale manifest. Refusing to log a bug because the
release block moved is disproportionate — the note gets lost instead of the schema getting
fixed.

It may do so only when all of these hold:

- the fields it actually reads are present and well-formed — `paths.work`, `ids.*`,
  `taxonomy.request_types`, `taxonomy.priorities`;
- it says once, briefly, that the manifest is on an older schema and `/work-migrate` will update
  it;
- it writes nothing outside `requests.md`.

If any field it needs is missing or malformed, it stops like everything else. No other command
gets this exemption — they all read blocks that have moved between eras.

## Using resolved values

After reading the manifest, treat it as the only source of project facts for the rest of
the session:

| Instead of | Use |
|---|---|
| a project name | `<project.name>` |
| `docs/work/requests.md` | `<paths.work>/requests.md` |
| `docs/spikes/` | `<paths.spikes>/` |
| `CHANGELOG.md` | `<paths.changelog>` (skip the step entirely if null) |
| `REQ-` / `WORK-` | `<ids.request_prefix>-` / `<ids.work_prefix>-` |
| a hardcoded agent name | a `name` from `<agents>` |
| "the human tags the release" | the behaviour implied by `<vcs.system>` and `<vcs.owner>` |
| a restated rule about what needs tests | read `<testing.policy_document>` |
| assuming the project is under git | `<vcs.system>` — it may be `none` |

## Optional path handling

`paths.architecture`, `paths.decisions`, `paths.product`, and `paths.changelog` may be
null. When a path is null, skip the step that would have used it and say so once. Do not
create the directory and do not treat its absence as an error.

## Inactive agents

Before routing work to any agent, check `<inactive_agents>`. If the target role appears
there, route to its `redirect_to` value instead and state why. Never spawn an agent that
is not in `<agents>`.
