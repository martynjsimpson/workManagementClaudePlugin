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

**First check whether one exists below.** A repository holding several projects — a monorepo
of apps, a workspace of packages — has a manifest per member and none at the top, so walking
up from the repository root correctly finds nothing while three manifests sit two levels
down. Before reporting an unmanaged repository, scan up to three levels below the current
directory for the same three candidates, excluding `node_modules`, `dist`, `build`, `.venv`,
`vendor`, and `target`.

If that finds one or more, **do not pick one.** Report what you found and where, and say the
session needs to start in the project's own directory:

> No manifest at this level. This repository holds N managed projects:
> `apps/toolA`, `apps/toolB`. Start the session in one of those directories, or `cd` there
> and re-run.

Selecting a member on the human's behalf is how work lands in the wrong project. There is no
`--project` flag and there should not be one: the working directory is already an
unambiguous statement of which member is being worked, and a flag would let it disagree
with the files being edited.

If the scan finds nothing either, stop. Report exactly:

> No work-management manifest found. Run `/work-init` to set this project up.

Do not guess paths. Do not create files. Do not fall back to conventional locations. A
missing manifest is the one condition under which every command except `/work-init`
refuses to act — silently inventing a location is how the drift this plugin exists to
prevent gets reintroduced.

## Schema compatibility gate

**The supported schema is `model_version: 4`.** Check this before reading any other field and
before taking any action. A command that proceeds against a schema it does not understand will
misread a key that has moved — silently, and with a plausible-looking result.

| Condition | Action |
|---|---|
| `model_version` == 4 | Continue to the shape cross-check below. |
| `model_version` < 4 | **Stop.** Report: "This manifest uses schema version N; this plugin needs version 4. Run `/work-init --upgrade` to update it." |
| `model_version` > 4 | **Stop.** Report that the manifest is newer than the plugin, and that the plugin should be updated. Do not proceed by ignoring unknown fields. |
| `model_version` absent | **Stop.** Treat as pre-4 and direct to `/work-init --upgrade`. |

**Never auto-migrate.** No command may rewrite the manifest as a side effect of doing something
else — a surprise nobody asked for, and the manifest is the one file meant to stay
human-authoritative. `/work-init --upgrade` (alias `--repair`) is the one deliberate exception:
running it *is* asking for the manifest to be brought current, and it still interviews the human
for every decision the schema change requires rather than filling one in silently. It performs
the same procedure `/work-migrate` describes, in the same session, before regenerating the agent
files that depend on the result. Run `/work-migrate` directly instead when you want to review or
control the manifest edit on its own, separate from agent regeneration.

**Shape cross-check.** A manifest can declare `4` and still be wrong — hand-edited, or
part-migrated. Confirm the current markers are present: a top-level `scope:` block with `root`,
`writes_outside` and `agents_dir`; a top-level `version:` block carrying `tag_template`; `vcs:`
with `system`; `release:` carrying only `changelog`, `pipeline`, `deploy_steps`; `testing:` with
`policy_document`; `excludes` on every agent. If the declaration and the shape disagree, say so
and stop. Do not repair it inline — except inside `/work-init --upgrade`, which exists for
exactly this.

For what each earlier era looks like and how it maps forward, read `schema-history.md`. Note
especially that `model_version: 2` covers four different shapes, because the number was left
unchanged across three breaking changes — so a version-2 manifest must be identified by its
keys, not by its declaration.

## Required fields

If a required field is missing (`project.name`, `project.description`,
`project.primary_reference`, `scope.root` — which may be null but must be present,
`paths.work`, `paths.spikes`, `ids.*`, `taxonomy.*`, `vcs.system`, `version.scheme`,
`version.owner`, `release.*`), name the missing field and stop. Do not substitute a default
for a required field.

A field being **present with a null value** is not the same as missing. `scope.root: null` is
an explicit statement that the project is the whole repository; an absent `scope` block is a
manifest that predates the question.

## The two roots

Every path in the manifest resolves against one of two roots, and confusing them is the
failure this section exists to prevent.

| Root | What it is | How to find it |
|---|---|---|
| **VCS root** | the directory containing `.git`. Where branch, tag, push and merge happen. | `git rev-parse --show-toplevel` |
| **project root** | where this project's world begins and ends | `<VCS root>/<scope.root>`, or the VCS root when `scope.root` is null |

When `<scope.root>` is null the two are the same directory and everything below collapses to
the behaviour that has always applied. When it is set — `apps/toolA` — they differ, and the
distinction is load-bearing.

### Path resolution

**Every path field in the manifest is relative to the project root.** That covers all of
`paths.*`, `project.primary_reference`, `project.domain_rules`, `version.file`,
`version.mirrors`, `testing.policy_document`, `release.pipeline.definition`, each agent's
`stack`, and every entry in an agent's `owns`, `excludes` and `reads`.

**A leading slash anchors to the VCS root instead**, exactly as it does in `.gitignore`:

| Manifest value | `scope.root: null` | `scope.root: apps/toolA` |
|---|---|---|
| `docs/work` | `docs/work` | `apps/toolA/docs/work` |
| `CHANGELOG.md` | `CHANGELOG.md` | `apps/toolA/CHANGELOG.md` |
| `/CHANGELOG.md` | `CHANGELOG.md` | `CHANGELOG.md` |
| `/.github/workflows/release.yml` | `.github/workflows/release.yml` | `.github/workflows/release.yml` |

Three fields are the exception and are **always** VCS-root-relative, with or without a leading
slash: `scope.root` itself, and every entry in `scope.writes_outside`. They describe where the
boundary sits, so they cannot be expressed relative to it.

The point of this rule is that a member's manifest reads exactly like a standalone project's.
Lift `apps/toolA` out into its own repository and the only line that changes is `scope.root`.

### The write boundary

**Never write outside the project root.** When `<scope.root>` is set, a write is permitted
only when its path resolves under the project root, or matches an entry in
`<scope.writes_outside>`. This binds every command in this plugin and every agent it spawns.

Reads are not fenced. Building against a shared package means reading it, and an allowlist for
reads would be pure friction.

If a task appears to require a write outside the boundary, **stop and say so** rather than
doing it. The fix is a `writes_outside` entry the human adds deliberately, not a judgement
call made mid-session. This matters most in the case the field exists for: a repository where
a sibling directory is another team's app, being worked in another checkout, right now.

## VCS scoping protocol

`<scope.root>` changes how git is used, not whether it is used. Git itself is unaffected by
the boundary — it walks up to find `.git` from any subdirectory, so a session started in
`apps/toolA` has full access to commit, tag, branch and push. What changes is the pathspec
every command must supply.

Skip this whole section when `<vcs.system>` is `none`, and when `<vcs.owner>` is `human`.

**Scope every status and diff check to the project.** A repo-wide `git status --porcelain`
reports another member's uncommitted work as this project's dirt, which turns a clean-tree
precondition into a blocker no one here can clear:

```
git status --porcelain -- <project root>
```

Where a check covers `<scope.writes_outside>` paths too, add them to the pathspec. Where
`<scope.root>` is null, the pathspec is the whole repository and the command is what it always
was.

**Stage by explicit path, never by sweep.** `git add -A`, `git add .`, `git add :/` and
`git commit -a` are forbidden outright when `<scope.root>` is set — each one stages another
member's work into this project's release. Name every path, and confirm each resolves under
the project root or matches `<scope.writes_outside>` before staging it.

**Tag with `<version.tag_template>`.** A tag name is repo-global. See the tag rules below.

**Branch names already carry the project slug** — `release/<project-slug>-<version>` — which
is what keeps two members' release branches from colliding on a shared remote. That was true
before `scope` existed and needs no change.

**Fetch before merging into a shared base.** Two checkouts of one repository — a second clone,
or a `git worktree` — mean the base branch may have moved since this checkout last saw it.
Fetch and report whether the base was behind, rather than merging onto a stale ref.

### Tag rules

Render the tag from `<version.tag_template>`:

- **null** — the version alone, per `<version.scheme>`: `v1.2.3` under `semver-v`, `1.2.3`
  under `semver`, `2026.07.27` under `date`. This is the historical behaviour.
- **set** — substitute `{version}` with that same rendering, and `{slug}` with
  `<project.name>` lower-cased with non-alphanumerics collapsed to single hyphens. So
  `"{slug}/v{version}"` on a project named ToolA gives `toola/v1.2.3`, and a literal
  `"toolA/v{version}"` gives `toolA/v1.2.3`.

**A null `tag_template` on a project with `<scope.root>` set is a warning, every time a tag is
about to be created.** It means this member is claiming the repository's global `v1.2.3`, and
the second member to reach that number cannot tag at all. Say so and offer to set the field;
do not refuse to tag, because the human may have a convention that genuinely works.

**Cross-check the tag against the pipeline trigger.** Where `<release.pipeline.definition>`
names a file and `<release.pipeline.trigger>` is `tag-push`, read the file and compare its tag
filter with the rendered tag. A workflow watching `v*` does not fire on `toolA/v1.2.3`. Report
a mismatch **before** tagging — a release that tags cleanly and silently builds nothing is
worse than one that fails loudly.

## The one exception: degraded mode for capture

The `work-capture` skill may proceed on a stale manifest. Refusing to log a bug because the
release block moved is disproportionate — the note gets lost instead of the schema getting
fixed.

It may do so only when all of these hold:

- the fields it actually reads are present and well-formed — `paths.work`, `ids.*`,
  `taxonomy.request_types`, `taxonomy.priorities`;
- an absent `scope` block is treated as `root: null`, which resolves `paths.work` exactly as
  the older schema did. This is safe here and only here: capture writes one file, at a path
  the older manifest already spelled correctly, so there is nothing for the boundary to
  protect that the path itself does not;
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
| assuming the project is the whole repository | `<scope.root>` — it may be one member of many |
| a bare `git status --porcelain` | the same scoped to the project root |
| `v<version>` as the tag | `<version.tag_template>`, rendered |

## Optional path handling

`paths.architecture`, `paths.decisions`, `paths.product`, and `paths.changelog` may be
null. When a path is null, skip the step that would have used it and say so once. Do not
create the directory and do not treat its absence as an error.

## Inactive agents

Before routing work to any agent, check `<inactive_agents>`. If the target role appears
there, route to its `redirect_to` value instead and state why. Never spawn an agent that
is not in `<agents>`.
