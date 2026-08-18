---
description: Update a project.yml written against an older plugin version to the current schema, preserving its comments and asking about anything that needs a decision.
argument-hint: "[--dry-run]"
---

Bring an existing manifest up to the current schema. This is the only command permitted to
rewrite `project.yml`, and it exists so that no other command has to guess at an older shape.

Most of the time you will not run this directly. `/work-init --upgrade` (alias `--repair`)
detects a stale manifest and performs this exact procedure automatically, in the same session,
before regenerating the agent files that depend on the result — that is the command to reach
for after bumping the plugin version. Run this command on its own when you want to review or
control the manifest edit by itself, separate from agent regeneration — for example to look at
the diff before anything downstream reads it.

Read `${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/schema-history.md` first. It defines
the current version, every earlier era with the keys that identify it, and the transition from
each.

## Step 0 — Locate and preflight

Find the manifest per
`${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/config-resolution.md`. **Ignore that
document's compatibility gate for this command only** — an out-of-date manifest is the input
here, not an error.

If no manifest exists, stop: there is nothing to migrate. Direct to `/work-init`.

**Preflight per `vcs.system`** — read it directly from the file, defaulting to a `.git` check if
the block does not exist (era A manifests have no `vcs:`):

- **git** — check the tree is clean with `git status --porcelain`, scoped to the directory
  holding the manifest where that is not the repository root. Read-only git only. A dirty
  tree is not a hard stop, but say the migration rewrites `project.yml` and a clean baseline
  makes the change reviewable as a single diff.
- **none** — copy the manifest to `.work-migrate-backup-<YYYYMMDD-HHMM>/` before writing.
  Report the location. There is no diff to fall back on.

**If `--dry-run` was passed**, write nothing. Run every step, print the era detected with its
evidence, the full proposed manifest, every decision that would need an answer, and the
comment-preservation report. Then stop.

## Step 1 — Detect the era by shape, not by declaration

**Do not trust `model_version`.** It was left at `2` across three breaking schema changes, so a
manifest declaring `2` may be any of four shapes. Identify the era from the keys actually
present, testing for the newest markers first, per `schema-history.md`.

Report the era you detected **and the evidence for it** — the specific keys present and absent.
If the shape does not match any era cleanly, say so and stop rather than guessing: a
part-migrated or hand-edited manifest needs a human eye, and a wrong guess here corrupts the one
file everything else derives from.

If the manifest is already current and its shape agrees, say so and stop. Nothing to do.

## Step 2 — Plan the transition

Work out the full set of changes from `schema-history.md`, and sort them into two kinds. The
distinction matters more than the mechanics:

**Mechanical** — a key renamed or moved with its value intact. `release.version_scheme` becomes
`version.scheme`. No judgement, no loss. Apply these without asking.

The **era E → F expansion is mechanical despite looking like a redesign.** `vcs.owner` and
`vcs.branching` become the six-entry `vcs.stages` map, and every one of their six combinations
has exactly one destination — the table is in `schema-history.md`, and no cell in it needs a
human decision. Add `vcs.delete_branch: after-merge`, which is what era E did unconditionally,
and rename `version.owner: command` to `agent`. Under `vcs.system: none`, set every stage to
`none` rather than applying the table.

**Show the expansion as a table in Step 3 even though it needs no decision.** The keys do not
merely move — one answer becomes six, and a human skimming a list of renames will not otherwise
see that "who owns git" is no longer a question this manifest asks. Point at the `pr` row
specifically if it applies: era E's open-the-PR-and-stop behaviour is now `merge: human` plus
`tag: agent`, with the tag deferral falling out of stage order rather than being stated.

**Needs a decision** — a new field with no correct default, or an old field whose replacement
means something different. Never fill these in silently. The known ones:

- **`scope.root`** (era D onward) — whether this project is the whole repository or one member
  of it. **Establish the evidence before asking.** Run `git rev-parse --show-toplevel` and
  compare it with the directory the manifest's project sits in:

  - **The manifest is at the repository root.** Propose `root: null` and move on. This is the
    overwhelmingly common case, it preserves era D behaviour byte for byte, and turning it
    into a question makes a non-decision look like one.
  - **The manifest is below the repository root.** This project has been managed as though it
    were a whole repository while actually being a member. Say what you found, name the
    proposed `scope.root`, and ask whether the repository holds other projects this one must
    not touch — because the answer is genuinely either way, and a project can live in a
    subdirectory of a repository it nonetheless owns entirely.

    Where the answer is yes, **the paths need stripping as well as the root adding.** Era D
    paths are repository-relative, era E paths are project-relative, so `paths.work:
    apps/toolA/docs/work` becomes `docs/work`. Present the full before/after list for every
    affected field — `paths.*`, `project.primary_reference`, `project.domain_rules`,
    `version.file`, `version.mirrors`, `testing.policy_document`,
    `release.pipeline.definition`, and every agent's `owns`, `excludes`, `reads` and `stack`
    — and get approval on the list as a whole. A path that does not start with the `scope.root`
    prefix is the interesting case: it points outside the new boundary and needs either a
    leading slash to anchor it to the VCS root (right for a shared CI workflow) or a
    `writes_outside` entry (right for a file the project genuinely writes). Ask which, per
    path. Do not strip silently and do not guess — a wrong root here breaks every ownership
    table at once.

- **`scope.writes_outside`** — empty unless the path review above turned up a file outside the
  new boundary that this project writes. Never add an entry speculatively.
- **`scope.agents_dir`** — `project` unless `scope.root` is null, where it is inert. Mention
  that the roster will move to `<scope.root>/.claude/agents/` when `/work-init --repair` next
  runs, and that agent names gain the project slug as a prefix.
- **`version.tag_template`** — `null` where `scope.root` is null; that is the historical
  behaviour and needs no thought. Where `scope.root` is non-null, **do not default it.** Run
  `git tag --list --sort=-creatordate`, show the recent tags, and ask. If they already carry a
  member prefix the answer is to match that convention. If they do not, say plainly that a
  bare `v1.2.3` is a repository-global name that the next member to reach that version cannot
  claim, and propose `{slug}/v{version}`.

  Where `release.pipeline.definition` names a workflow with `trigger: tag-push`, read it and
  compare its tag filter against what the proposed template renders, in the same exchange. A
  workflow watching `v*` will not fire on `toolA/v1.2.3`.

- **`version.mirrors`** — which other version-bearing files a release should keep in step. Search
  the tree for every `package.json`, `pyproject.toml`, `Cargo.toml`, `*.csproj` and `VERSION`
  file, excluding dependency and build directories. Present the list with each current value and
  ask which belong. An empty list is a valid answer and must be an explicit one.
- **`excludes` on agents** — adding `excludes: []` everywhere is mechanical, but then compute
  effective ownership and check for overlap. Where an overlap exists, propose the carve-out on
  both sides and confirm it. An era B manifest often has a comment describing the overlap as
  "most-specific-path-wins" or similar; that comment is the signal a carve-out needs declaring,
  and it should be replaced by the declaration it was standing in for.
- **`testing.policy_document`** (era A only) — the old `policy` / `mandatory_areas` /
  `condition_of_done` fields restated a codebase fact. The replacement points at where that fact
  properly lives. Ask which document states the testing policy. `null` is valid; say what it
  costs. **Preserve the old values in your report** so the human can paste them into that
  document — this is the one transition that discards content rather than moving it.
- **`vcs.system`** (era A only) — establish it by checking for a repository, then confirm.
- **`vcs.stages` beyond the mechanical expansion** — nothing here needs asking, but *offer*.
  Once the six fields exist, per-stage ownership is expressible for the first time, and the
  arrangement the old two fields forced may not be the one the human would choose. Show the
  expanded table and ask, once, whether any stage should change now that it can — for example
  cutting the branch themselves while the agent commits and opens the pull request, or setting
  `tag: none` on a project that carries its version in a file and never tags. Accept "leave it"
  without pressing; the expansion is already correct.

## Step 3 — Present the plan

Show, before writing:

- the era detected and the evidence;
- every mechanical change as `old.key -> new.key`, with the value carried;
- every decision, with your recommendation and the options;
- **every path whose resolution changes**, as a before/after list, where `scope.root` ends up
  non-null. This is the one change a reader cannot infer from the key names, because the keys
  do not move — their meaning does;
- **the stage expansion as a six-column table**, where `vcs.owner` and `vcs.branching` were
  present. One answer becoming six is not legible as a list of renames;
- any field being removed, and where its content should go instead;
- the backup location under `vcs.system: none`.

Get explicit approval. Get separate answers to each decision — do not bundle them into one
confirmation, because `version.mirrors` in particular is a per-project judgement that a blanket
"yes" would paper over.

## Step 4 — Edit the file textually, preserving comments

**Edit the existing file in place. Do not parse the YAML and re-serialise it.**

This is the most important instruction in this command. A manifest that has been in use carries
hand-written comments explaining *why* a project is configured as it is — why ADRs live inline
rather than in a directory, why a role is deliberately absent, what an empty list means. Round
-tripping through a YAML library destroys every one of them and leaves a file that parses
identically and has lost all its reasoning. That reasoning is the manifest's main value over a
pile of defaults.

So:

- move and rename keys as text edits, carrying any comment attached to a key along with it;
- keep every existing comment, including ones that now sit in a different block;
- add the template's explanatory comment for each genuinely new key, so a reader can tell what
  it is for;
- update comments that have become factually wrong — a reference to `release.version_file`
  should become `version.file` — but do not rewrite a human's prose to suit your own phrasing;
- delete a comment only when it describes a workaround the migration has just made unnecessary,
  and say in the report that you removed it and why;
- set `model_version` to the current value last, so a failure partway through leaves the file
  declaring its old version rather than falsely claiming to be current.

## Step 5 — Verify

Re-read the file and confirm:

- it parses as valid YAML;
- it satisfies the shape cross-check in `config-resolution.md`;
- every value from the original is present somewhere — compare key by key against what you read
  in Step 1, and account for anything that has gone;
- **every path resolves to a file or directory that exists**, applied against the project root
  per the new rules. This catches a mis-stripped prefix immediately, where leaving it would
  surface later as an agent that mysteriously owns nothing;
- comment count has not dropped except where Step 4 authorised a removal;
- no `TODO` or placeholder remains where a decision was supposed to be answered.

If verification fails, say exactly what failed and leave the backup in place. Do not attempt a
second pass over a file you have already half-edited.

## Step 6 — Report and follow up

- era detected, and the version now declared;
- mechanical changes applied;
- decisions taken, with the answers given;
- content removed and where it now needs to live — particularly the era A testing fields;
- comments preserved, added, and removed;
- backup location under `vcs.system: none`, with a reminder to delete it once satisfied.

Then name the follow-up: **run `/work-init --repair` (or `--upgrade`, the same flag).** The
generated agent files carry scope tables derived from the manifest, so a migration that adds
`excludes` or changes ownership leaves those files stale until they are regenerated. A migration
is not finished until that is done, and saying so is part of this command's job.

## Constraints

- Write only to `project.yml` and the backup directory. Nothing else.
- Never parse-and-redump the YAML. Textual edits only.
- Never invent a value for a field that needs a human decision.
- Never migrate as a side effect of another command; this command must be invoked deliberately.
- Never guess an era. An unrecognised shape is a stop, not a best effort.
- Do not touch `requests.md`, `backlog.yml`, `active-release.md`, or any agent file —
  `/work-init --repair` regenerates agents afterwards.
- Do not run mutating git commands.
- Under `--dry-run`, write nothing.
