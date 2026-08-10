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

**Establish whether the project is under version control.** Check for a `.git` directory,
then confirm with `git rev-parse --is-inside-work-tree`. Read-only git only — never `add`,
`commit`, `checkout`, `stash`, or anything else that mutates state.

**If it is a git repository**, check the working tree is clean with
`git status --porcelain`:

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
- Record the answer for the manifest: `vcs.system: none`, with `vcs.owner` and
  `vcs.branching` set to `null`. Do not set `system: git` on the assumption that version
  control will be added later.

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
- list every file you would **create**, **modify in place**, **regenerate over**, and **skip**
  — as four separate lists, not one;
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

- a dependency manifest (`package.json`, `pyproject.toml`, `go.mod`, `*.csproj`,
  `Cargo.toml`, `pom.xml`);
- **every version-bearing file** — search the whole tree, not just the root. Look for a
  `version` field in any `package.json`, `pyproject.toml`, `Cargo.toml`, or `*.csproj`, and
  for any plain `VERSION` file. Exclude `node_modules`, `dist`, `build`, `.venv`, and
  anything else the project ignores.

  **Do not stop at the first hit.** A workspace monorepo has a manifest per member —
  `frontend/`, `backend/`, `shared/` — and treating the root as the only one is wrong in
  both directions: it misses files a release should bump, and it silently assumes members
  track the root version when they may be independent or unversioned.

  Record every file found and its current value. You will ask which is the version of
  record and which, if any, are mirrors;
- the top-level source directories;
- an existing docs tree, and a changelog;
- **a pipeline definition** — `.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`,
  `azure-pipelines.yml`, `.circleci/config.yml`, `bitbucket-pipelines.yml`. Where you find
  one, read it: identify what triggers it (a tag push, a branch push, manual dispatch) and
  what it produces. Offer the file path as `release.pipeline.definition` rather than
  transcribing its behaviour into the manifest — the path stays accurate as the workflow
  evolves, a transcription does not;
- **a testing policy statement** — check `README.md`, `TECHNICAL_ARCHITECTURE.md`,
  `AI.md`, `CLAUDE.md`, `CONTRIBUTING.md`, and any `docs/testing*`. You are looking for
  where the project already says what needs test coverage;
- a data-model or schema file;
- any existing `.claude/agents/` files.

Note what you found — you will offer it as defaults.

Then ask about the things you cannot reliably infer. Use one multiple-choice question per
topic where the options are genuinely distinct, and offer detected values as the
recommended option. Do not ask about anything the survey already answered
unambiguously.

Cover, in this order:

1. **Project name and one-line description.** The description goes into every generated
   agent prompt, so it must say what the thing *is*, not what it does well.
2. **Paths.** Where the work files, spikes, architecture docs, decision records, product
   docs, and changelog live. Offer the detected docs layout. Any of architecture,
   decisions, product, and changelog may be null.
3. **The product truth source.** The single document agents align planned work to. If
   none exists, say so plainly — the model works without one, but `/work-plan` will have
   nothing to check scope against, and that is worth the human knowing now.
4. **ID prefixes.** Default `REQ` and `WORK`.
5. **Type vocabulary.** Offer the model's default sets. If work files already exist,
   extract the types actually in use and make the manifest a superset — do not propose a
   vocabulary that would immediately fail validation.
6. **Version control.** If Step 0 found a git repository, ask who runs commit/tag/push
   (`vcs.owner`) and the branching model (`vcs.branching`).

   When explaining this, note that the release coordinator is a persona `/work-release`
   adopts, not a sub-agent — `vcs.owner: command` means that persona does the VCS work.

   **When Step 0 established there is no version control**, set `vcs.system: none`, set
   `vcs.owner` and `vcs.branching` to `null` rather than leaving their defaults in place, and
   skip both questions entirely — they have no meaning.

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
globs against the tree. A typo silently produces an agent that owns nothing, and an
`excludes` typo silently hands a path back to the wrong owner.

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

For each entry in `<agents>`, write `.claude/agents/<name>.md` from the matching template:

- `product-manager` -> `${CLAUDE_PLUGIN_ROOT}/templates/agents/product-manager.md`
- `principal-architect` -> `${CLAUDE_PLUGIN_ROOT}/templates/agents/principal-architect.md`
- everything else -> `${CLAUDE_PLUGIN_ROOT}/templates/agents/implementer.md`

For each entry in `<inactive_agents>`, write `.claude/agents/<name>.md` from
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
- Read-only git only — `status`, `rev-parse`, `log`. Never `add`, `commit`, `checkout`,
  `stash`, `reset`, or any other mutating command.
- Under `--dry-run`, write nothing whatsoever.
- Do not proceed past an ownership overlap.
- Do not invent an agent, a status, or a path that the human did not confirm.
