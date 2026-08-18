---
description: Set up, repair, or upgrade the work-management model in this repository — write the manifest, scaffold the work files, generate the agent roster, and bring a stale manifest current first if needed.
argument-hint: "[--dry-run] [--repair|--upgrade]"
---

Set this repository up to use the work-management model. Everything project-specific ends
up in one manifest; the commands and this plugin stay untouched.

Read `${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/model.md` before writing
anything, so the scaffolded files match the model exactly.

This command runs against repositories that already have work in them. Treat existing
content as the thing to be protected, not as an obstacle.

## Step 0 — Preflight

**Establish whether the project is under version control.** Run
`git rev-parse --is-inside-work-tree`, then `git rev-parse --show-toplevel` to find the VCS
root. Read-only git only — never `add`, `commit`, `checkout`, `stash`, or anything else that
mutates state. Do not test by looking for a `.git` directory in the current folder: a project
inside a monorepo legitimately has its repository several levels above, and that test reports
"no version control" on a repository that plainly has one.

**Establish the project boundary before anything else.** Compare the VCS root with the
directory this command is running in.

- **Same directory** — the ordinary case. The project is the repository, `scope.root` will be
  null, and nothing else in this step changes.
- **Different** — the current directory sits inside a larger repository. Do not assume either
  way what that means; a project can legitimately live in a subdirectory of a repository it
  nonetheless owns entirely. Look for siblings: list the entries beside the current directory
  and check whether any is a plausible peer project — its own dependency manifest, its own
  source tree. Then ask:

  > This directory sits inside a repository rooted at `<VCS root>`. Is `<current dir>` the
  > whole of what I should manage, or does the repository hold other projects I must not
  > touch?

  Their answer sets `scope.root`: the path from the VCS root to the current directory when
  the project is one member of several, null when the repository is theirs entirely and they
  just happen to be standing in a subdirectory of it.

Carry the answer through the whole session. Where `scope.root` will be set, **every survey,
every scan and every git command below is scoped to the project root** — read the two-roots
and VCS-scoping sections of
`${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/config-resolution.md` now rather than
improvising the scoping per step.

**If it is a git repository**, check the working tree is clean with `git status --porcelain`
— scoped to the project root where `scope.root` will be set, so another member's uncommitted
work is not reported as this project's:

- **Clean:** continue, and tell the user that everything this command writes will therefore
  show as a reviewable diff they can discard wholesale.
- **Dirty:** stop. List the uncommitted paths and say why this matters — this command writes
  a manifest, up to five scaffold files, and one file per agent, and without a clean
  baseline there is no way to separate its output from work in progress. Offer to continue
  anyway only if the user explicitly asks, and record that they were warned.

**If it is not under version control**, this is a supported configuration, not a problem to
route around. Say so plainly, then compensate for the missing safety net:

- State clearly that there is **no undo**. Nothing this command writes can be reverted by
  discarding a diff.
- **Take a backup before writing anything.** Back up **every existing file this run will
  modify or generate over** — not only the agent files. That means, at minimum: the manifest
  itself, if Step 1 finds one and Step 1a will edit it; each agent file you will extract from
  or regenerate; `<version.file>` and every path in `<version.mirrors>` if you will initialise
  or change a version; the `<project.domain_rules>` file if you will append to it; and any
  existing `stack` file you will append to. A file being *created* needs no backup — deleting
  it is a clean undo. Getting this wrong is how an in-place edit to a dependency manifest
  becomes unrecoverable.
- **Put the backups in one timestamped directory, not beside the originals.** A sibling
  `.bak` file inside a directory that gets scanned for definitions becomes a phantom
  definition — `.claude/agents/backend-developer.md.bak` will be read as an agent. Use a
  single directory such as `.work-init-backup-<YYYYMMDD-HHMM>/` outside `.claude/`,
  mirroring the original paths inside it. Report the exact location.
- Note that the extract-then-generate step in Step 5 is the riskiest thing this command
  does, and recommend `--dry-run` first.
- Record the answer for the manifest: `vcs.system: none`, with every entry in `vcs.stages`
  set to `none`. Do not set `system: git` on the assumption that version control will be
  added later.

Under `git`, a clean tree makes backups unnecessary — the diff is the backup. Do not create
backup files on a git project; they add clutter that then shows up in the very diff the user
is meant to review.

Do not treat the absence of git as a reason to refuse. Plenty of projects — prototypes,
personal tools, anything on a single machine — are legitimately not under version control,
and the model works perfectly well without it. What changes is the safety strategy and
where the durable record of a release lives.

**If `--dry-run` was passed**, write nothing at all for the whole session. Run every step
normally up to the point of writing, then instead:

- print the manifest you would write, in full;
- state the project root, the VCS root, and the roster directory the manifest resolves to —
  as literal paths, so a wrong `scope.root` is visible here rather than after every file has
  been written against it;
- list every file you would **create**, **modify in place**, **regenerate over**, and **skip**
  — as four separate lists, not one, each as a full path from the repository root;
- state where backups would go and which files they would cover;
- render **one complete agent file** — pick the one with the most complex scope table — so
  the user can see actual generated output rather than a summary of it;
- report any preflight, overlap, or extraction problem you found.

Then stop and say that nothing was written. A dry run that writes "just the manifest" is
not a dry run.

## Step 1 — Detect existing state

Look for an existing manifest using the discovery order in
`${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/config-resolution.md`.

- **No manifest:** continue to Step 2.
- **Manifest exists:** before anything else, run the schema compatibility gate from
  `config-resolution.md` — read `model_version` and run the shape cross-check. Do this
  regardless of whether `--repair`/`--upgrade` was passed. Regenerating agents or offering the
  menu below off a manifest whose keys have moved is exactly the silent-misread failure the
  gate exists to prevent, and this command is not exempt from it just because it is also the
  one that can fix it.

  - **Newer than this plugin understands:** stop. Report that the manifest is newer than the
    plugin and the plugin needs updating. Do not proceed by ignoring unknown fields.
  - **Stale** (`model_version` below current, absent, or declared current but the shape
    cross-check disagrees): do not fall into either branch below. Go to
    **Step 1a — Upgrade a stale manifest** instead.
  - **Current and shape matches:** continue to whichever of the next two branches applies.

- **Manifest exists, current schema, `--repair`/`--upgrade` not passed:** report where it is
  and what it configures, then ask whether to (a) regenerate the agent files from it, (b)
  validate the work files against it, or (c) stop. Do not overwrite a manifest that already
  exists.
- **Manifest exists, current schema, `--repair` or `--upgrade` passed:** skip to Step 5,
  regenerating agents and re-validating without touching the manifest. Note that this only
  regenerates files carrying the `generated-by` marker. On a first adoption no file carries
  one, so it will correctly skip every hand-written agent — say so explicitly rather than
  reporting a silent no-op, because the expected outcome was regeneration.

`--repair` and `--upgrade` are the same flag under two names, with identical behaviour.
`--upgrade` reads better right after bumping the plugin version; `--repair` reads better when
the schema hasn't changed and you just need agent files regenerated or fixed.

### Step 1a — Upgrade a stale manifest

Say plainly what is happening and why: this manifest was written against an older version of
the plugin, and every other command has already been refusing to touch it rather than risk
misreading a moved key. Bringing it current is a prerequisite for everything else this command
does — including plain `--repair`'s agent regeneration, since a scope table derived from a
stale manifest would be wrong in the same silent way.

Say that this performs the same procedure `/work-migrate` describes, in this session, so
nothing is left half-done waiting on a second command the user has to remember to run
separately. Then get one confirmation to proceed with the upgrade — not per-decision yet, just
permission to start.

If they decline, stop cleanly. Say that `/work-migrate` exists standalone for running just the
manifest edit on its own — for example to review that diff before touching agent files — and
that this command can be re-run to finish the job once that is done.

If they confirm, perform `${CLAUDE_PLUGIN_ROOT}/commands/work-migrate.md` Steps 1 through 5
exactly as written there: detect the era, plan the transition, present the plan and get
approval on each decision separately, edit the manifest textually, and verify. Use this
command's own Step 0 preflight (already run) rather than work-migrate's — do not preflight
twice or take a second backup.

**Under `--dry-run`:** follow work-migrate's own `--dry-run` behaviour for this half — print
the detected era with evidence, the full proposed manifest, and every decision that would need
an answer, writing nothing — then continue into this command's own `--dry-run` reporting in
Step 5 for the regeneration half. Nothing gets written for either half.

Once verification passes and the manifest is confirmed current, continue directly into this
command's Step 5 to regenerate the agent files against the now-current manifest. Report the
migration and the regeneration together at the end in Step 6 — this is one operation from the
user's perspective even though it followed two specifications to get there.

## Step 2 — Interview

This step is reached only from Step 1's "No manifest" branch — every other branch skips
straight to Step 5. Before asking anything, survey the repository so the interview asks
about real things rather than hypotheticals. Look for:

**Scope every search below to the project root**, not the VCS root, where Step 0 established
a boundary. Surveying the whole monorepo turns a ten-item question into a hundred-item one and
invites the human to pick another member's file by mistake. The two exceptions are called out
where they arise: the pipeline definition and the tag convention are repository-wide facts and
must be looked for above the boundary.

- a dependency manifest (`package.json`, `pyproject.toml`, `go.mod`, `*.csproj`,
  `Cargo.toml`, `pom.xml`);
- **every version-bearing file** — search the whole project tree, not just its top level.
  Look for a `version` field in any `package.json`, `pyproject.toml`, `Cargo.toml`, or
  `*.csproj`, and for any plain `VERSION` file. Exclude `node_modules`, `dist`, `build`,
  `.venv`, and anything else the project ignores.

  **Do not stop at the first hit.** A workspace has a manifest per member —
  `frontend/`, `backend/`, `shared/` — and treating the top level as the only one is wrong in
  both directions: it misses files a release should bump, and it silently assumes members
  track the top-level version when they may be independent or unversioned.

  Record every file found and its current value. You will ask which is the version of
  record and which, if any, are mirrors;
- the top-level source directories;
- an existing docs tree, and a changelog;
- **a pipeline definition** — `.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`,
  `azure-pipelines.yml`, `.circleci/config.yml`, `bitbucket-pipelines.yml`. **Look at the VCS
  root as well as the project root**: CI configuration is usually repository-wide even when
  the project is not, and a monorepo member's release almost always runs from a workflow
  defined above it. Where the definition sits outside the project root, offer it with a
  leading slash — `/.github/workflows/release.yml` — so it resolves against the VCS root.

  Where you find one, read it: identify what triggers it (a tag push, a branch push, manual
  dispatch) and what it produces. On a shared workflow, note **which tag or branch patterns it
  filters on** — that is what Step 2's tag question has to match. Offer the file path as
  `release.pipeline.definition` rather than transcribing its behaviour into the manifest —
  the path stays accurate as the workflow evolves, a transcription does not;
- **the repository's tag convention**, where `scope.root` will be set. Run
  `git tag --list --sort=-creatordate` and look at the most recent twenty. If they carry a
  member prefix — `toolA/v1.2.3`, `toolA-v1.2.3` — that is the convention this project must
  follow, and you will offer it as `version.tag_template` rather than inventing one;
- **a testing policy statement** — check `README.md`, `TECHNICAL_ARCHITECTURE.md`,
  `AI.md`, `CLAUDE.md`, `CONTRIBUTING.md`, and any `docs/testing*`. You are looking for
  where the project already says what needs test coverage;
- a data-model or schema file;
- any existing agent files — check `.claude/agents/` at the project root and, where a boundary exists, at the VCS root too, since an earlier setup may have put them either side of it.

Note what you found — you will offer it as defaults.

Then ask about the things you cannot reliably infer. Use one multiple-choice question per
topic where the options are genuinely distinct, and offer detected values as the
recommended option. Do not ask about anything the survey already answered
unambiguously.

Cover, in this order:

1. **Project name and one-line description.** The description goes into every generated
   agent prompt, so it must say what the thing *is*, not what it does well.

1a. **The boundary**, but only where Step 0 established one. Skip this group entirely when
   `scope.root` will be null — on an ordinary repository there is nothing here to decide, and
   asking makes a non-question look like a decision.

   Confirm `scope.root` as the path Step 0 arrived at, then ask the two things that follow
   from it:

   - **Anything outside the boundary this project must be able to write.** Default to nothing,
     and mean it: this is the field that keeps a release from touching a sibling app, so an
     entry added "just in case" removes the protection the block exists for. A root
     `CHANGELOG.md` the project appends to, or a shared registry file it must register itself
     in, are the kinds of thing that qualify. Reading is not restricted, so do not collect
     read paths here — say that explicitly, because the natural instinct is to list every
     shared package the project builds against.
   - **Where the agent roster should live** (`scope.agents_dir`). Recommend `project` —
     `<scope.root>/.claude/agents/` — with the reason: it keeps the member self-contained, and
     it is found when the session starts in the project directory, which is how a monorepo
     member is meant to be worked. Offer `repo` for a human who always starts Claude Code at
     the repository root instead. Say plainly that this is about **where the files are
     discovered from**, and if the roster later fails to load, this is the field to change.

   Then state, once, the thing that most needs saying and is easiest to get wrong: **git does
   not need the session to start at the repository root.** It walks up to find `.git` by
   itself, so working from `<scope.root>` gives full commit, branch, tag and push while
   keeping every file operation inside the project. Starting at the repository root to "get
   git working" is the mistake this whole block exists to make unnecessary.

2. **Paths.** Where the work files, spikes, architecture docs, decision records, product
   docs, and changelog live. Offer the detected docs layout. Any of architecture,
   decisions, product, and changelog may be null.

   **Where `scope.root` is set, offer these relative to the project root** —
   `docs/work`, not `apps/toolA/docs/work`. Say so when you present them, because the
   detected paths were found as full repository paths and a human reading `docs/work` on a
   monorepo will reasonably wonder which `docs/work` is meant. Use a leading slash only for
   something that genuinely lives above the boundary, such as a repository-wide changelog.
3. **The product truth source.** The single document agents align planned work to. If
   none exists, say so plainly — the model works without one, but `/work-plan` will have
   nothing to check scope against, and that is worth the human knowing now.
4. **ID prefixes.** Default `REQ` and `WORK`.
5. **Type vocabulary.** Offer the model's default sets. If work files already exist,
   extract the types actually in use and make the manifest a superset — do not propose a
   vocabulary that would immediately fail validation.
6. **Release stages.** If Step 0 found a git repository, settle `<vcs.stages>` — the six
   things a release does, each saying separately whether it happens and who does it. Read
   the release-stages section of `config-resolution.md` before asking, so the vocabulary and
   the validity rules are in hand.

   **Do not ask six questions.** Offer named shapes and let one answer write all six fields,
   then show the resulting table for confirmation. The shapes worth offering:

   | Shape | `branch` | `commit` | `push` | `merge` | `pull_request` | `tag` |
   |---|---|---|---|---|---|---|
   | Trunk — commit and tag on the current branch | `none` | `agent` | `agent` | `none` | `none` | `agent` |
   | Release branch, merged locally | `agent` | `agent` | `agent` | `agent` | `none` | `agent` |
   | Release branch, pull request | `agent` | `agent` | `agent` | `none` | `agent` | `agent` |
   | You cut branches, the agent does the rest | `human` | `agent` | `agent` | `none` | `agent` | `agent` |
   | You own the repository entirely | `human` | `human` | `human` | `human` | `none` | `human` |

   These are an interview convenience, not a manifest field. Always write the six values out
   explicitly — a preset key in the schema would re-bundle exactly what the stage table exists
   to unbundle. Say that any shape can be adjusted stage by stage, and adjust it there and then
   if they want to.

   **Ask about tagging separately if they take a shape that tags.** `tag: none` is a perfectly
   ordinary answer — plenty of projects carry the version in a file and never tag — and it is
   the one stage a preset should not quietly decide. Where they choose `none`, confirm
   `version.file` will be set, since the version then has nowhere else to live.

   When explaining any of this, note that the release coordinator is a persona `/work-release`
   adopts, not a sub-agent — `agent` names who acts, the AI rather than them, and never
   authorises spawning a sub-agent for it.

   **Validate before writing.** Run the whole validity table from `config-resolution.md`
   against whatever they chose, and report every failure at once. Flag interleaved ownership —
   an `agent` stage between two `human` ones — as a session that will stall mid-release rather
   than as an error; it is legitimate, and the cost is worth knowing now.

   Then ask `<vcs.delete_branch>`, but only where a `merge` stage is active: `after-merge` is
   the historical behaviour and the default, `never` keeps the release branch.

   **When Step 0 established there is no version control**, set `vcs.system: none`, set every
   stage to `none`, and skip this group entirely — the questions have no meaning.

7. **Versioning.** Ask this as its own group. Versioning and version control are independent
   concerns and must not be bundled: a project can carry a version in `package.json` with no
   repository at all, or sit under git with the version expressed only as a tag. Bundling them
   is how a project ends up unable to describe its own arrangement.

   - **Scheme** (`version.scheme`) and **who assigns the number** (`version.owner`).
   - **Where the version lives.** Show every version-bearing file the survey found, with its
     current value, as a list. Then ask two things: which is the version of record
     (`version.file`), and which of the others a release should keep in step
     (`version.mirrors`).

   Do not collapse the second question into the first or answer it yourself. On a workspace
   monorepo the right answer is genuinely project-specific — some members track the root
   version, some are `private` with a placeholder version that must not be touched. Present
   the list and let the user say. If they choose no mirrors, confirm that releases will touch
   only the one file, so the choice is on the record rather than a default nobody saw.

   - **The tag name** (`version.tag_template`), where `vcs.system` is `git`. Skip this on a
     project whose `scope.root` is null unless the survey found a tag convention that a bare
     version would break — the default renders `v1.2.3` and that is nearly always right.

     **Where `scope.root` is set, ask it every time and do not offer null as the
     recommendation.** A tag name is repository-global: the moment two members both reach
     1.2.3, the second cannot tag. Offer the convention the survey found in the existing tag
     list as the recommended option, since a repository that already tags `toolA/v1.2.3` has
     settled this question and the answer is to match it. Where no convention exists, propose
     `{slug}/v{version}` and say what it will render as.

     **Then cross-check the pipeline, in the same exchange.** If
     `release.pipeline.definition` names a workflow with `trigger: tag-push`, compare its tag
     filter against what the chosen template renders. A workflow watching `v*` will not fire
     on `toolA/v1.2.3`. Where they disagree, say so before the answer is recorded and name
     both fixes — widen the workflow's filter, or match the template to it — because a
     release that tags cleanly and builds nothing is the one failure here nobody notices until
     much later.

   Mention the timing, because it affects the answer: `/work-release` writes the version at the
   **start** of a release, right after scope and number are confirmed, so everything built
   during the session already carries it.

   **When `vcs.system` is `none`**, `version.file` becomes **required** — with no tag there is
   nowhere else for the version to live. If the survey found no version field anywhere, ask
   where it should live and offer either the dependency manifest's `version` field or a new
   plain `VERSION` file.

8. **Changelog and deploy steps.** Whether a changelog is required (`release.changelog`), and
   what manual steps the human performs after the release is out (`release.deploy_steps`).

   When `vcs.system` is `none`, recommend `changelog: required` and say why in one line: with
   no commit history it is the only narrative record that a release happened. Accept `optional`
   if the user insists, but say what they are giving up.
9. **Pipeline.** If the survey found a workflow definition, offer its path and the trigger
   you read from it, and confirm. If it found none, ask whether a pipeline exists elsewhere;
   the human may know of one the survey cannot see. If there is genuinely none, set the
   whole `pipeline` block to null — `/work-release` then commits, tags, pushes, and says
   plainly that no automated build follows.
   Record `pipeline.does` as free text ONLY when a pipeline exists but has no definition
   file in this repository. A prose description sitting alongside a definition file is a
   second copy, and it will drift.
10. **Testing.** Do not ask what needs test coverage — that is a property of the codebase and
   belongs with the project's technical documentation, not in this manifest. Instead, offer
   the testing policy document the survey found and confirm the path, or ask which document
   states the policy. If the project has no written testing policy, set
   `policy_document: null` and say plainly that verification will then rest entirely on each
   release's verification bar. Also ask whether a dedicated testing agent exists, or whether
   implementers write their own tests.
11. **Agent roster.** Propose one agent per top-level source area found in the survey,
   plus a product-manager and, if the project has architecture or decision docs, a
   principal-architect. For each: name, role title, model, and the paths it owns.
   Ownership must not overlap.

   **Where `scope.root` is set**, two things differ. Express `owns`, `excludes` and `reads`
   relative to the project root — `src`, not `apps/toolA/src` — so the roster reads the same
   as it would in a standalone repository. And **prefix every agent name with the project
   slug**: `toola-backend-developer`, not `backend-developer`. Two members of one repository
   will otherwise both propose a `backend-developer`, and the failure that produces is work
   routed to the other app's agent, which is confusing to diagnose from the symptom. Say you
   are doing this and why; do not present it as a naming preference.

   A shared package the project builds against belongs in `reads` with a leading slash —
   `/packages/shared` — not in `owns`. Owning it would claim a directory another member
   almost certainly also uses.
12. **Deliberately absent roles.** Ask which common roles do *not* exist here
    (test-engineer and devops-engineer are the usual candidates), and where work aimed at
    each should be routed instead. This is not busywork: unless a role is explicitly
    declared absent, agents invent it and route work into a role with no owner.
    Do not offer `release-manager` here. Release coordination is a persona `/work-release`
    adopts, so it is never an agent on any project — listing it as *absent* would imply it
    could have been present.
13. **Human-owned responsibilities.** What no agent may do.

## Step 3 — Check ownership, then confirm the plan

**Ownership overlap is a hard stop, not a warning.** Two agents owning the same path
produces two contradictory scope tables, each telling the other to keep out, and the
resulting deadlock is hard to diagnose from the symptom.

Compute effective ownership as `owns` minus `excludes`, expanding globs, and treating a
parent path as covering its children — `backend` covers `backend/src`. Then:

- **If two agents' effective ownership overlaps, stop.** Show the conflicting pairs.
- **Do not resolve it yourself by convention.** "Most-specific-path-wins" is a plausible
  rule and the wrong move: nothing downstream enforces it, so a path added under the
  narrower claim later will silently belong to two agents again. Fix it in the manifest
  where a check can see it.
- **Offer the carve-out explicitly.** Where the overlap is a genuine slice-inside-a-directory
  arrangement — a test role owning test files under a developer's tree — propose the
  `excludes` entries on both sides and confirm them. That turns a convention into a declared,
  checkable fact.
- **An excluded path must be owned by someone.** If a path is excluded by one agent and owned
  by none, that is a gap, not a carve-out. Report it and resolve it before continuing.

Also check that every path in `owns`, `excludes`, and `reads` actually exists, resolving
globs against the tree. **Resolve them against the project root, and a leading slash against
the VCS root**, per the two-roots rules in `config-resolution.md`. A typo silently produces an
agent that owns nothing, and an `excludes` typo silently hands a path back to the wrong owner.

**Then check the boundary**, where `scope.root` is set:

- No `owns` entry may escape the project root unless the same path appears in
  `scope.writes_outside`. An owned path outside the boundary that nothing authorised is not a
  configuration nuance — it is a write that will be refused at the moment it is attempted,
  halfway through a release. Stop and resolve it here.
- Every `scope.writes_outside` entry must resolve, and must sit outside `scope.root`. An entry
  already inside the boundary is redundant, and reads to the next person like a permission
  that was needed once.
- `reads` entries outside the boundary are fine and need no authorisation. Say so if the user
  asks — the asymmetry is deliberate.

Then present a compact summary before writing, split into three explicit lists so nothing
modifies a file the user did not expect:

- **Create** — new files and directories.
- **Modify** — existing files you will change in place: version fields, the domain-rules
  file, any existing stack file. Name every one.
- **Regenerate over** — hand-written agent files, contingent on the Step 5 extraction.
- **Skip** — files left alone because they already exist.

Plus the manifest values, and ownership as a table showing effective ownership after
`excludes`. Under `vcs.system: none`, also state the backup location and confirm the backup
set covers everything in the Modify and Regenerate lists.

Get explicit approval. Approval of this summary is not approval of an extraction — that is
asked for separately in Step 5.

## Step 4 — Write the manifest and scaffold

Copy `${CLAUDE_PLUGIN_ROOT}/templates/project.yml` to `<paths.work>/project.yml` and fill
in every answer. Keep the explanatory comments — they are what stops the next person
adding a project fact to a command.

Then create, only where the file does not already exist:

- `<paths.work>/requests.md` from `${CLAUDE_PLUGIN_ROOT}/templates/requests.md`
- `<paths.work>/backlog.yml` from `${CLAUDE_PLUGIN_ROOT}/templates/backlog.yml`
- `<paths.work>/active-release.md` from `${CLAUDE_PLUGIN_ROOT}/templates/active-release.md`
- `<paths.work>/README.md` from `${CLAUDE_PLUGIN_ROOT}/templates/work-README.md`
- `<paths.spikes>/` as a directory — but see the check below first

Substitute manifest values into every template placeholder. Generate the `requests.md`
legend from the manifest's status set and type vocabulary rather than copying a fixed
list.

Never overwrite an existing work file. If one exists, leave it and report that you did.

### Also permitted in this step, when the manifest calls for it

These writes are authorised, and each must appear in the Step 3 plan and — under
`vcs.system: none` — in the Step 0 backup set:

- **Create `<paths.changelog>`** if `<release.changelog>` is `required` and the file does not
  exist. Write a minimal header only; do not invent historical entries for releases you have
  no record of.
- **Initialise the version** in `<version.file>` and `<version.mirrors>` if
  those files exist but carry no version field. State the value you are setting and why. Do
  not change a version that is already present — that is a release action, not a setup
  action.
- **Append to `<project.domain_rules>`** with invariants extracted in Step 5, under a clearly
  titled heading. Append only; never rewrite what is already in the file.

Anything beyond this list is out of scope for `/work-init`. If the setup appears to need a
further write, surface it as a recommendation instead of performing it.

### Before creating the spikes directory

Check whether the project's own documentation says spikes are deliberately not used — an
architecture document or decision record stating the project does not keep spike documents.
If it does, do not create the directory silently. Surface the contradiction, and ask whether
to amend that decision or leave `<paths.spikes>` unscaffolded until a spike is actually
raised.

Scaffolding a directory a documented decision says should not exist is a small thing that
quietly undermines the document, and the next reader cannot tell which is current.

## Step 5 — Generate the agent roster

**Resolve the roster directory first.** Where `<scope.root>` is null it is `.claude/agents/`
at the repository root, as it has always been. Where `<scope.root>` is set, it follows
`<scope.agents_dir>`: `project` gives `<scope.root>/.claude/agents/`, `repo` gives
`.claude/agents/` at the VCS root. Every path below means that directory.

Under `agents_dir: repo` on a monorepo, that directory is shared with every other member, so
**never regenerate over a file whose agent name is not in this manifest's `<agents>` or
`<inactive_agents>`** — it belongs to a sibling project. The slug prefix from Step 2 is what
keeps the two sets distinct; if you find an unprefixed file whose name collides with one you
are about to write, stop and report it rather than overwriting another project's roster.

For each entry in `<agents>`, write `<roster dir>/<name>.md` from the matching template:

- `product-manager` -> `${CLAUDE_PLUGIN_ROOT}/templates/agents/product-manager.md`
- `principal-architect` -> `${CLAUDE_PLUGIN_ROOT}/templates/agents/principal-architect.md`
- everything else -> `${CLAUDE_PLUGIN_ROOT}/templates/agents/implementer.md`

For each entry in `<inactive_agents>`, write `<roster dir>/<name>.md` from
`${CLAUDE_PLUGIN_ROOT}/templates/agents/inactive-agent.md`.

**Derive the scope-enforcement table — never hand-write it.** For each agent, build the
table by inverting the ownership map: every path owned by a *different* agent becomes a
row redirecting to that agent, and every entry in `<human_owns>` becomes a row
redirecting to the human. This is the whole reason ownership is declared once. Adding an
agent later means editing the manifest and re-running `/work-init --repair`, not editing
every agent file.

Leave the stack and domain-rules sections as links to the files named in each agent's
`stack` field and in `<project.domain_rules>`. Do not inline stack detail into the
generated prompt — the prompt is the portable part, the stack is not.

### Existing hand-written agent files: extract before you generate

An existing agent file that lacks the `generated-by` marker is hand-written, and it almost
certainly contains project knowledge the manifest has no field for — interface conventions,
response envelope shapes, validation rules, domain formulas, UI consistency rules,
architectural invariants. The manifest deliberately does not hold these; it points at a
`stack` file and a `domain_rules` file instead.

**On a fresh adoption those target files do not exist yet.** So generating over a
hand-written agent file destroys content whose replacement home has not been created. Never
do that. Work in this order:

1. **Read the existing file and classify every section**, out loud, into three buckets:
   - *derivable* — ownership, scope tables, git and testing constraints, boilerplate. The
     generated file reproduces these from the manifest, so they can be dropped.
   - *stack and conventions* — technology choices, directory layouts, interface and naming
     conventions, framework-specific rules. These belong in the agent's `stack` file.
   - *domain invariants* — business formulas, calculation rules, cross-cutting product
     laws. These belong in `<project.domain_rules>`.

2. **Show the user that classification and get approval on it.** Do not proceed on your own
   reading. Something you judged boilerplate may be the one line that stops a recurring
   mistake.

3. **Write the extracted content to its new home first.** Create the `stack` file and the
   `domain_rules` file, with the content preserved close to verbatim — you are relocating
   it, not rewriting it, and paraphrasing loses the specificity that made it useful. If the
   manifest has no `stack` path for that agent or no `domain_rules` path, stop and get one;
   do not invent a location silently.

4. **Verify the new files exist and contain the content**, then generate the agent file.

5. **Report what moved, from where, to where** — line counts included, so the user can see
   nothing evaporated.

If the user declines the extraction, do not generate over the file. Leave it untouched,
report it as skipped, and note that its agent will not have a derived scope table until it
is either extracted or replaced. A hand-written agent file that still works is strictly
better than a generated one missing half its knowledge.

Where a `stack` or `domain_rules` file already exists, append to it rather than replacing
it, and flag any content that appears to contradict what is already there rather than
silently picking a side.

## Step 6 — Validate and report

Run the conformance checks listed in the `work-model` skill against what now exists.

Confirm before reporting success that no generated file still contains a `{{` placeholder,
and that every file you claimed to write actually exists on disk. Report:

- **if Step 1a ran** — the era detected, mechanical changes applied, decisions taken with the
  answers given, and anything removed, exactly as work-migrate.md's own Step 6 specifies, ahead
  of everything below;
- the manifest location and the key decisions it records;
- **the boundary**, where `<scope.root>` is set: the project root, the repository root, what
  `<scope.writes_outside>` permits, and where the agent roster landed. Add the one operational
  consequence in a line — future sessions should start in the project directory, and git works
  fine from there;
- files created, files skipped because they already existed;
- **content extracted** — what moved out of each hand-written agent file, to where, with
  line counts, so the user can confirm nothing evaporated;
- agent files generated, agent files left untouched and why, and the ownership table;
- roles declared absent and where their work routes;
- any conformance failure, with the fix.

Then, per `<vcs.system>`:

- **`git`** — tell the user to review the diff before committing. Everything written is new
  or extracted content, so `git diff` plus `git status` shows the whole change and
  discarding it is a clean revert.
- **`none`** — name the backup location from Step 0 again, and tell them to delete the
  backups once they are satisfied. Do not leave stale `.bak` files as the only record of
  what the originals contained.

Finish by naming the next step: `/work-plan` to refine intake, or the `work-capture`
skill to start capturing requests.

## Constraints

- Write only to: the manifest; the four work scaffold files; the generated agent files; the
  `stack` and `domain_rules` files you extract into; `<paths.changelog>` if creating it; and
  the version fields listed in `<version.file>` / `<version.mirrors>` if
  initialising them. Nothing else.
- Do not overwrite an existing manifest or work file, except the Step 1a upgrade of a stale
  manifest — and even there, edit it textually per work-migrate.md's method, never parse and
  re-serialise the YAML.
- Do not change a version that already has a value.
- Do not touch a version-bearing file that is not named in the manifest, even if the survey
  found it.
- Do not generate over a hand-written agent file until its extracted content exists in its
  new home and the user has approved the classification.
- Read-only git only — `status`, `rev-parse`, `log`, `tag --list`. Never `add`, `commit`,
  `checkout`, `stash`, `reset`, or any other mutating command.
- Never write outside the project root once `<scope.root>` is established, and never propose
  a `writes_outside` entry the human did not ask for. Every write in the list above resolves
  under the project root by construction; if one appears not to, that is the signal a path
  answer was given as a repository path where a project path was wanted.
- Do not proceed past an `owns` entry that escapes the boundary without a matching
  `writes_outside` entry.
- Under `--dry-run`, write nothing whatsoever.
- Do not proceed past an ownership overlap.
- Do not invent an agent, a status, or a path that the human did not confirm.
