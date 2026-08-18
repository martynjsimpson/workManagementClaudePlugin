# Template placeholder reference

How `/work-init` fills each `{{PLACEHOLDER}}`. Values in `<angle brackets>` are manifest
lookups from `<paths.work>/project.yml`.

**Resolve every path before substituting it.** Manifest paths are relative to the project
root, with a leading slash anchoring to the VCS root — see the two-roots section of
`skills/work-model/references/config-resolution.md`. A generated agent file carries resolved
paths, never the manifest's shorthand: the manifest is written to stay portable, and the
prompt is written to be acted on.

Placeholders marked **derived** must be computed, not asked about. They are the reason
ownership and release mechanics are declared once — computing them is what keeps N agent
files consistent with one manifest.

## Direct substitutions

| Placeholder | Source |
|---|---|
| `{{PROJECT_NAME}}` | `<project.name>` |
| `{{PROJECT_DESCRIPTION}}` | `<project.description>` — reads as an appositive after the project name, so phrase it "a local-first web app for X", not "This project does X". |
| `{{PRIMARY_REFERENCE}}` | `<project.primary_reference>` |
| `{{WORK_PATH}}` | `<paths.work>` |
| `{{SPIKES_PATH}}` | `<paths.spikes>` |
| `{{REQUEST_PREFIX}}` | `<ids.request_prefix>` |
| `{{REQUEST_TYPES}}` | `<taxonomy.request_types>`, comma-separated |
| `{{WORK_TYPES}}` | `<taxonomy.work_types>`, comma-separated |
| `{{PRIORITIES}}` | `<taxonomy.priorities>`, comma-separated |
| `{{AGENT_NAME}}` | the agent's `name` |
| `{{AGENT_ROLE}}` | the agent's `role` |
| `{{AGENT_MODEL}}` | the agent's `model` |

## Derived — ownership

**`{{PROJECT_BOUNDARY_SECTION}}`** (derived) from `<scope>`:

- **`scope.root` is null** — omit entirely, heading and all. The project is the repository and
  there is no second boundary to describe. Do not emit a paragraph saying so; a generated
  agent that opens by explaining a constraint that does not apply to it has spent its most
  valuable lines on nothing.
- **`scope.root` is set** — a short paragraph, before the owned-paths list, stating the
  project root as a literal path and that nothing outside it may be written. Name the
  repository root too, and say that reading outside is fine. Where `<scope.writes_outside>`
  has entries, list them as the named exceptions with whatever reason the manifest's comments
  give. For example:

  > **This project is `apps/toolA`, one member of a repository rooted at
  > `internal-tools/`.** Everything you write goes inside `apps/toolA`. You may read anything
  > in the repository — a shared package, a root config — but the directories beside yours
  > belong to other projects and are not yours to change, even to fix something obviously
  > broken in them. Report it instead.

  The ordering matters: this comes *before* the owned-paths list because it is the outer
  constraint, and the paths below it are a subdivision of what it permits.

**`{{BOUNDARY_CONSTRAINT}}`** (derived) — the one-line constraint form of the same fact.

- **`scope.root` is null** — emit `You do not write outside the repository.` The line is
  nearly free and it forecloses an agent reaching into a sibling directory on the filesystem.
- **`scope.root` is set** — `You do not write outside `<scope.root>`. Other directories in
  this repository belong to other projects; report problems there rather than fixing them.`
  Append the `<scope.writes_outside>` entries as the exceptions where any exist.

**`{{OWNED_PATHS_LIST}}`** — a markdown list of the agent's `owns` entries, one per line, each
as `` - `path` `` followed by a short gloss of what lives there if the survey established it.

Where `<scope.root>` is set, **render these as full paths from the repository root** —
`apps/toolA/src`, not `src` — even though the manifest stores them project-relative. The
manifest is optimised for portability; a generated prompt is optimised for an agent that has
to act on it, and `src` inside a monorepo is ambiguous in exactly the way that matters.

**`{{OWNED_PATHS_PROSE}}`** — the same paths as a comma-separated inline phrase, for the
frontmatter `description`.

**`{{OWNERSHIP_EXCLUSIONS}}`** (derived) — if the agent has `excludes` entries, a short
paragraph naming them and the agent that owns each instead. For example:

> Except: test files, the harness in `backend/src/test/`, and the Vitest configs sit inside
> your directories but belong to the **test-engineer**.

Put this immediately after the owned-paths list, before the read-only section — an exclusion
is a correction to the claim just made, and it needs to be read alongside it rather than
discovered further down in the scope table.

If `excludes` is empty, omit this placeholder's content entirely.

**`{{READ_PATHS_SECTION}}`** — if the agent has `reads` entries, a paragraph naming them as
read-only. If `reads` is empty, omit the whole section including its heading.

**`{{SCOPE_TABLE}}`** (derived) — build a two-column markdown table, `| Requested task |
Redirect to |`, by inverting the *effective* ownership map (`owns` minus `excludes`, globs
expanded):

- one row per path owned by a *different* agent -> redirect to that agent's name in bold;
- one row per entry in this agent's own `excludes` -> redirect to whichever agent owns it.
  These rows matter more than the rest of the table: an excluded path sits *inside* a
  directory the agent otherwise owns, so it is the boundary the agent is most likely to cross
  without noticing;
- one row per entry in `<human_owns>` -> redirect to **the human**;
- one row per entry in `<inactive_agents>` whose work might plausibly land here -> redirect to
  that entry's `redirect_to`, noting the role does not exist;
- for the architect's `consult_before` triggers, one row on every *other* agent's table
  redirecting to the architect;
- where `<scope.root>` is set, **one row as the first in the table**, covering everything
  outside the project root at once:

  | Change anything outside `apps/toolA` | **The human** — that is another project in this repository |

  Do not enumerate the sibling directories. There may be twenty, the list goes stale the day
  someone adds the twenty-first, and the catch-all is both shorter and correct by
  construction. Where `<scope.writes_outside>` has entries, name them in this row as the
  exceptions rather than giving each its own.

**One exception: never emit a row that blocks `<paths.spikes>`.** Spike output is a defined
deliverable of a spike work item, and `suggested_agents` may assign a spike to any agent. If
the spikes directory appears in one agent's `owns` list, a naive inversion would forbid every
other agent from writing the document it was assigned to produce. Instead, add a row to every
agent's table reading:

| Write a spike document for a spike work item assigned to you | Allowed — `<paths.spikes>/<ITEM-ID>.md`, despite the ownership above |

Never hand-write this table and never copy it between agents. Adding an agent to the manifest
and re-running `/work-init --repair` must be sufficient to update every file.

Do not emit a row for a `release-manager` or `release-coordinator`. That role is a persona
`/work-release` adopts and is never an agent, so there is nothing to redirect to.

## Derived — release and testing

**`{{VCS_CONSTRAINT}}`** (derived) from `<vcs>`:

- `system: none` -> `This project is not under version control. Do not run git commands, and
  do not ask anyone to commit, tag, or branch — there is no repository. Changes you make take
  effect immediately and cannot be reverted, so confirm anything destructive first.`
- `system: git`, `stages.commit: human` -> `You do not commit, branch, tag, or push. The human
  performs every version-control operation on this project, including committing your work.`
- `system: git`, `stages.commit: agent` -> `You do not commit. The /work-release command
  commits your work as part of the release.`

**Resolve this from `<vcs.stages.commit>` alone.** It is the only stage an implementer's work
depends on — whether the human or the release coordinator later pushes, merges or tags changes
nothing about what the implementer does, and enumerating the other five in a generated prompt
is noise the agent has no use for.

In no case does the implementing agent run VCS commands. What differs is who does, or whether
anyone can — and stating it explicitly prevents the agent from inventing an answer, which on a
project with no repository means inventing a repository.

The no-VCS wording carries a second job: it tells the agent its edits are unrecoverable. That
should make it more cautious about destructive changes than it would be with a clean tree
behind it.

**`{{VERSION_CONSTRAINT}}`** (derived, product-manager only) from `<version.owner>`:

- `human` -> `You do not assign version numbers. /work-plan always writes Version: TBD; the
  human supplies the number at the start of the release session.`
- `agent` -> `You do not assign version numbers. /work-plan always writes Version: TBD;
  /work-release assigns it at the start of the release session.`

Either way the PM writes `TBD`. What differs is who fills it in, and when — which is always at
the start of `/work-release`, not at ship time.

**`{{TESTING_SECTION}}`** (derived) from `<testing>`:

- **`policy_document` is set** -> instruct the agent to read that document and treat it as the
  authority on what requires test coverage. **Link to it; do not summarise it.** Summarising
  creates a second copy of a codebase fact in a generated file, which is exactly the drift the
  manifest exists to prevent — and the generated file is the copy nobody thinks to update.
- **`policy_document` is null** -> state that the project has no written testing policy, so
  the condition of done is whatever the work item's acceptance criteria and the release's
  verification bar specify. Do not invent a policy to fill the gap.

If `<testing.agent>` names an agent, add that tests are written by it; otherwise state that the
implementing developer writes their own.

**`{{CONSULT_TARGET_SENTENCE}}`** (derived) — if an architect exists in `<agents>`, name it and
list its `consult_before` triggers inline. If none exists, write: `This project has no
architect — raise the decision with the human instead.`

## Derived — architect sections

**`{{CONSULT_TRIGGERS_LIST}}`** — the architect's `consult_before` entries as a markdown list.

**`{{DECISIONS_SECTION}}`** — if `<paths.decisions>` is set, instruct that decisions are
recorded there, numbered sequentially from the last existing record, each stating context,
decision, consequences, and alternatives considered. If null, instruct that decisions are
recorded in the relevant architecture document, and note the project has no decision-record
directory.

**`{{ARCHITECTURE_SECTION}}`** — if `<paths.architecture>` is set, name it. If null, omit the
maintenance instruction and say the project keeps no separate architecture documents.

## Derived — stack and domain

**`{{STACK_SECTION}}`** — if the agent's `stack` field names a file, link to it and instruct the
agent to read it before implementing. If null, write a short stack summary from the `/work-init`
survey, and add a note that moving this into a dedicated file keeps the generated prompt short.

Do not inline a long stack table into the generated prompt. The prompt is the portable part; the
stack is not, and a stack table in a generated file is a fact that will go stale where nobody
looks for it.

**`{{DOMAIN_RULES_SECTION}}`** — if `<project.domain_rules>` names a file, link to it and
instruct the agent to treat its contents as invariants. If null, omit the section and its
heading entirely rather than emitting an empty one.

## Derived — inactive agents

| Placeholder | Source |
|---|---|
| `{{INACTIVE_REASON}}` | the entry's `reason`, as a complete sentence |
| `{{REDIRECT_TO}}` | the entry's `redirect_to` |

## Rules

1. Every placeholder must be substituted. A generated file containing `{{` is a bug — check for
   this before reporting success.
2. Omit optional sections entirely, including their headings, rather than emitting a heading
   with "none" or "N/A" beneath it.
3. Keep the `<!-- generated-by: -->` marker. `/work-init --repair` uses it to tell generated
   files from hand-written ones, and will not overwrite a file that lacks it.
