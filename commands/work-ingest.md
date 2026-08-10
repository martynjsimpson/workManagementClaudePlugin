---
description: One-time backfill of requests.md from existing project material — a PRD, TODO or roadmap files, code comments, or a superseded planning folder.
argument-hint: "[--dry-run] [source-path]"
---

Backfill `requests.md` from work that already exists in the project in some other form. This
is a content operation, not a configuration one: `/work-init` gives you a correctly
structured but empty intake file, and this fills it.

Designed to be run more than once — once per source, as you get to them. It is not a
migration you must complete in a single pass.

## Step 0 — Resolve configuration and preflight

Follow `${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/config-resolution.md`. Read the
manifest at `<paths.work>/project.yml`. Refuse to proceed if it is missing — run `/work-init`
first.

Read `${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/model.md` for the request format.

**Preflight per `<vcs.system>`:**

- **`git`** — check the working tree is clean with `git status --porcelain`. Read-only git
  only. A dirty tree is not a hard stop here, but say that this command appends to
  `requests.md` and may move source files, so a clean baseline makes the change reviewable.
- **`none`** — there is no undo. Back up `<paths.work>/requests.md` before writing if it
  already has content, into `.work-ingest-backup-<YYYYMMDD-HHMM>/`. Say where.

**If `--dry-run` was passed**, write nothing and move nothing for the whole session. Run every
step, then print: the sources found, the full text of every request you would write, the
already-built items you would omit with the evidence for each, the duplicates you would skip,
and the source files you would retire. Then stop and say nothing was written.

## Step 1 — Identify sources

If a path was given as an argument, use only that. Otherwise sweep for work hiding in other
forms, in roughly this order of yield:

| Source | What to look for |
|---|---|
| `<project.primary_reference>` | requirements and capabilities described as intended behaviour |
| `TODO.md`, `ROADMAP.md`, `BACKLOG.md`, `IDEAS.md`, `NOTES.md` | anything listed as wanted or outstanding |
| A superseded planning folder | phase, sprint, or milestone documents with unfinished items |
| `TODO`, `FIXME`, `HACK`, `XXX` comments in source | defects and deferred work, excluding dependencies and build output |
| README and architecture docs | "known limitations", "future work", "not yet implemented" sections |
| Skipped tests | `.skip`, `.todo`, `xit`, `pending` — each is usually a known gap |
| `<paths.changelog>` | an Unreleased section describing work in flight |

**Classify each source as one of two kinds — this determines what happens to it in Step 7:**

- **Work-only** — the file exists solely to capture outstanding work. `TODO.md`,
  `ROADMAP.md`, `BACKLOG.md`, a superseded planning folder. Its content is fully replaced by
  the requests you extract.
- **Mixed-content** — the file has a purpose beyond capturing work. The PRD, a README, an
  architecture document, the changelog, any source code file. Extracting from it removes
  nothing.

Present the sources found, with the kind and a rough candidate count for each, and ask which
to ingest. Do not ingest everything found without asking — a PRD and a stale planning folder
often describe the same work, and ingesting both produces duplicates that are tedious to
unpick afterwards.

## Step 2 — Extract candidates

**Granularity: one request per capability, with the individual requirements preserved as
prose in `Summary`.**

A PRD section normally becomes one request, not fifteen. Splitting into work items is
`/work-plan`'s job, and doing it here produces a `requests.md` nobody reads. But do not
summarise away the detail on the way through — carry the specific requirements into the
`Summary` so nothing is lost before refinement.

For each candidate:

- **Title** — short and specific. "Transaction categorisation", not "Categorisation feature".
- **Summary** — what is wanted, then the individual requirements as prose. Preserve concrete
  values, formats, and constraints verbatim: a threshold, a date format, a named third party.
  These are exactly what gets lost in paraphrase and exactly what an implementer needs.
- **Type** and **Priority** — from `<taxonomy.request_types>` and `<taxonomy.priorities>`
  only. Where the source states a priority, use it. Where it does not, use `medium` and do not
  pretend to more knowledge than the source gave you.
- **Status** — always `inbox`. Everything ingested is unreviewed by definition.
- **Source** — cite the origin precisely: `docs/product/PRD.md §4.2`,
  `backend/src/services/import.ts:214`, `TODO.md line 31`. Cite rather than restate; the
  source document stays authoritative and `requests.md` stays small.

## Step 3 — Verify each candidate against the codebase

**This is the step that makes the difference between a useful backlog and a misleading one.**

A PRD describes the intended product. On an established project much of it already exists.
Ingesting it naively produces requests for features that shipped months ago — worse than an
empty file, because it looks authoritative and the first `/work-plan` session will start
refining work that is already done.

For each candidate, establish whether the capability exists. Derive search terms from the
candidate — entity names, route paths, component names, distinctive UI labels — then search
the source tree, the tests, and `<paths.changelog>`. Where there are many candidates, dispatch
the verification across parallel read-only subagents by source section; it is a fan-out search
and doing it serially is slow enough that the temptation to skip it becomes real.

Classify into exactly one of four:

**Built** — the capability demonstrably exists. **Omit it.** Do not create a request, and do
not create a `done` request either: a completion record with no version behind it is
fabricated history, and `/work-prune` will later have nothing to verify it against. Record it
in the report with the evidence that satisfied you, so the judgement is visible and
challengeable.

**Partially built** — some of it exists. **Ingest it**, and make the `Summary` state what
exists and what is missing. These are the highest-value requests the whole command produces,
because the gap is the thing nobody has written down anywhere.

**Absent** — no trace. Ingest as `inbox`.

**Unverifiable** — you could not establish it either way in reasonable effort. Ingest as
`inbox` and add a `Notes:` line saying existence was not verified and why. Do not resolve an
unverifiable candidate by guessing in either direction; an honest "unverified" costs one
refinement question, a wrong guess costs a wasted work item or a lost requirement.

State your evidence in the report for every Built classification. This is the one place where
being wrong silently loses a requirement, so the reasoning has to be inspectable.

## Step 4 — Deduplicate

Check every surviving candidate against:

- existing requests in `<paths.work>/requests.md`, all four sections including `Done` and
  `Deferred / rejected` — a candidate matching a rejected request should not be silently
  re-added;
- existing work items in `<paths.work>/backlog.yml`;
- other candidates in this same run, since two sources frequently describe one thing.

Where a candidate duplicates an existing request, skip it and report the match. Where it adds
detail the existing request lacks, propose appending to that request's `Notes:` instead of
creating a new one.

## Step 5 — Present for approval

Work in batches of no more than fifteen, grouped by source. Do not present eighty requests at
once — nobody reviews that, and unreviewed bulk ingestion is how the backlog becomes something
to be ignored.

For each batch show: the full request text, and for each item its classification with a
one-line justification. Get approval for the batch before writing it. Accept edits to any
request before it is written.

Also present, once, before the first batch:

- the count of candidates omitted as already built, with the evidence for each;
- the count skipped as duplicates, with what they matched;
- anything you classified unverifiable, and why.

## Step 6 — Write the requests

Allocate IDs from `<ids.request_prefix>` and `<ids.pad>`, continuing from the highest existing
ID. Never reuse an ID even where a gap exists — a gap usually means something was pruned.

Append to `## Inbox / needs refinement` in the exact format from the model: an H3 heading with
the ID, then plain `Key: value` lines. No bullets, no nested headings, no YAML.

Write batch by batch as each is approved, not all at the end. A failure partway through then
leaves approved work on disk rather than losing all of it.

## Step 7 — Retire work-only sources

Once the requests from a source are written and verified on disk, offer to retire that source.

**Only work-only sources may be retired.** Move the file into
`.work-ingest-backup-<YYYYMMDD-HHMM>/`, mirroring its original path inside that directory, so
`requests.md` becomes the single place outstanding work lives. This is the point of retiring
it: leaving both means two lists that immediately start diverging.

**Never move a mixed-content source.** Specifically, never move:

- `<project.primary_reference>` — the manifest points at it and the PM reads it every planning
  session. Moving it breaks the configuration.
- any path named anywhere in `<paths>` or elsewhere in the manifest;
- a README, architecture document, or changelog;
- any source code file. A `TODO` comment is cited, never excised.

Before moving anything, confirm three things and say so:

1. Every request from that source is present in `requests.md`.
2. The user has approved the move for that specific file.
3. Content from the file that was **not** ingested — candidates omitted as already built — will
   be archived with it. Name what those were, because if a Built classification was wrong, the
   backup folder is where the evidence now lives.

Under `<vcs.system>` `git`, note the move shows as a rename in the diff. Under `none`, note the
backup directory is the only remaining copy and should be kept until they are satisfied.

Never delete a source file. Moving is reversible; deleting is not.

## Step 8 — Report

- requests written, by source, with ID ranges;
- candidates omitted as already built, with the evidence for each;
- duplicates skipped and what they matched;
- items marked unverifiable, needing a human answer at refinement;
- sources retired and where they went; sources deliberately left in place and why;
- any source found but not ingested this run, so the remaining work is visible.

Finish by pointing at `/work-plan` to refine the new inbox, and note that a large ingest is
usually worth refining in several sessions rather than one.

## Constraints

- Everything ingested is `Status: inbox`. Never write directly to `backlog.yml` — that skips
  refinement, which is the step that makes a request implementable.
- Never create a `done` request for work you believe already shipped. Omit and report it.
- Never move or delete a mixed-content source, a manifest-referenced path, or a code file.
- Never delete any file.
- Do not modify `<paths.work>/project.yml`, `active-release.md`, or `backlog.yml`.
- Do not invent a type, priority, or status outside the manifest's vocabulary.
- Do not present more than fifteen requests in one batch.
- Under `--dry-run`, write nothing and move nothing.
