# Manifest schema history

The current schema is **`model_version: 5`**. A plugin refuses to act on any other value; see
`config-resolution.md`.

## The honest caveat

`model_version` was left at `2` across three breaking schema changes (plugin 0.3, 0.4 and 0.6).
So **a manifest declaring `model_version: 2` may be any of four different shapes.** The number
cannot be trusted for those files.

Consequence for migration: `/work-migrate` must **detect the shape** of a version-2 manifest
from the keys actually present, not from what it claims. From version 3 onward the number is
reliable and detection is a belt-and-braces cross-check rather than the primary mechanism.

## Eras

Each era lists the keys that identify it. Detect by testing for the newest tells first.

### Era F — `model_version: 5` (plugin 1.4+) — current

- `vcs:` holds `system`, a `stages:` map, and `delete_branch` — and **no** `owner` or
  `branching`
- `vcs.stages` carries all six of `branch`, `commit`, `push`, `merge`, `pull_request`, `tag`,
  each valued `none | agent | human`
- `version.owner` takes `human | agent`; the value `command` no longer exists anywhere
- everything era E had, unchanged

**What changed in meaning:** there is no longer a global "who owns version control" answer.
`vcs.owner` asked once what `vcs.stages` now asks six times, and `vcs.branching` bundled
*whether* each stage happens with *which* stages happen at all. Splitting them is what makes
"you cut the branch, the agent commits and opens the PR, nobody tags" expressible.

Tag timing became derived rather than declared. Because `tag` follows integration in the stage
order, a `human`-owned `merge` already defers the tag to a later run — so no separate timing
field is needed.

### Era E — `model_version: 4` (plugin 1.3)

**Tells:** `scope:` block present; `vcs:` still carries `owner` and `branching`; no
`vcs.stages`.

- top-level `scope:` block with `root`, `writes_outside`, `agents_dir`
- `version:` block carries `tag_template` alongside `scheme`, `owner`, `file`, `mirrors`
- everything era D had, unchanged

Every path field is relative to the **project root** rather than the repository root, with a
leading slash escaping to the VCS root. On a manifest whose `scope.root` is null those are the
same directory, so **no existing path needs rewriting** — which is what made D → E an additive
migration rather than a rewrite.

**To reach F:** expand the two old keys into the stage map. This is fully mechanical — every
combination has exactly one destination, and none of it needs a human decision:

| `owner` | `branching` | `branch` | `commit` | `push` | `merge` | `pull_request` | `tag` |
|---|---|---|---|---|---|---|---|
| `command` | `none` | `none` | `agent` | `agent` | `none` | `none` | `agent` |
| `command` | `branch` | `agent` | `agent` | `agent` | `agent` | `none` | `agent` |
| `command` | `pr` | `agent` | `agent` | `agent` | `human` | `agent` | `agent` |
| `human` | `none` | `none` | `human` | `human` | `none` | `none` | `human` |
| `human` | `branch` | `human` | `human` | `human` | `human` | `none` | `human` |
| `human` | `pr` | `human` | `human` | `human` | `human` | `human` | `human` |

Then: add `vcs.delete_branch: after-merge`, which is what era E did unconditionally; and rename
`version.owner: command` to `agent`. Under `vcs.system: none`, set every stage to `none`
instead of applying the table.

The `command`/`pr` row is the one worth reading twice. Era E opened the PR and stopped without
tagging, expecting a re-run after the merge — that is `merge: human` plus `tag: agent`, and the
deferral falls out of the ordering rather than needing to be stated.

### Era D — `model_version: 3` (plugin 0.6–1.2)

**Tells:** top-level `version:` block present but **no** `tag_template` inside it; **no**
top-level `scope:` block.

- top-level `version:` block with `scheme`, `owner`, `file`, `mirrors`
- `vcs:` with `system`, `owner`, `branching`
- `release:` with only `changelog`, `pipeline`, `deploy_steps`
- `testing:` with `policy_document`, `agent`
- every entry in `agents:` has `excludes`

**To reach F:**

- add `scope:` with `root: null`, `writes_outside: []`, `agents_dir: project`. `root: null`
  preserves era D behaviour exactly, so this is the safe default and the only one that needs
  no thought on an ordinary single-project repository.
- add `version.tag_template: null` — likewise the historical behaviour.
- **Ask before defaulting `scope.root`, but only where the evidence warrants it.** If the
  manifest sits somewhere other than the repository root — the `.git` directory is above
  `<paths.work>`'s parent — this is a monorepo member that has been managed as though it
  were a whole repository, and its paths are almost certainly carrying an `apps/toolA/`
  prefix that should now be stripped. Say what you found, propose `scope.root` and the
  prefix-stripping as one change, and let the human confirm. Do not perform it silently: a
  path rewrite based on a wrong guess about the root breaks every ownership table at once.
- **Recommend a `tag_template` whenever `scope.root` ends up non-null**, and say why in one
  line: a bare `v1.2.3` is a repository-global name that the next member to reach that version
  cannot also claim.

### Era C — declared `2`, actually plugin 0.4–0.5

**Tells:** `release.version_mirrors` present; `excludes` present on agents; no top-level
`version:`.

**To reach F:** create the `version:` block and move four keys into it —
`release.version_scheme` → `version.scheme`, `release.version_owner` → `version.owner`,
`release.version_file` → `version.file`, `release.version_mirrors` → `version.mirrors`. Then
apply the D → E and E → F transitions above.

### Era B — declared `2`, actually plugin 0.3

**Tells:** `vcs:` present; `release.version_scheme` / `version_owner` / `version_file` present;
**no** `release.version_mirrors`; **no** `excludes` on any agent.

**To reach F:** the era C moves (which chain through D → E → F), plus:

- add `version.mirrors` — **requires a human decision.** Never default it silently; see the
  migration command.
- add `excludes: []` to every agent, then check for overlap. Where a genuine carve-out exists
  (typically a test role owning test paths inside a developer's tree), propose the `excludes`
  entries on both sides. Era B manifests frequently document this overlap in a comment as
  "most-specific-path-wins" — that comment is the signal a carve-out needs declaring.

### Era A — declared `2`, actually plugin 0.1–0.2

**Tells:** **no** `vcs:` block at all; `release.git_owner` and `release.branching` present;
`testing.policy`, `testing.mandatory_areas`, `testing.condition_of_done` present.

**To reach F:** the era B and C moves (which chain through D → E → F), plus:

- create `vcs:` — `release.git_owner` → `vcs.owner`, `release.branching` → `vcs.branching`, and
  set `vcs.system` by checking whether the project is actually under git.
- rework `testing:` — replace `policy`, `mandatory_areas` and `condition_of_done` with a single
  `policy_document` pointing at the project's own testing documentation. **Requires a human
  decision**: the old fields restated a codebase fact, and the new field points at where that
  fact properly lives. If no such document exists, `null` is valid but the human should know
  what they are giving up.
- `release.pipeline` may be a plain string in era A; convert to the
  `{definition, trigger, does}` map, preferring a path to a workflow file over the prose.

## Rules for adding an era

1. Bump `model_version` in `templates/project.yml`.
2. Add the era here with its tells and its transition to current.
3. Teach `/work-migrate` the transition, including which new fields need a human decision
   rather than a default.
4. Update the supported version in the `work-model` skill and in `config-resolution.md`.
5. Never silently accept an old shape by making the new field optional — that is how
   `model_version` became decorative in the first place.
