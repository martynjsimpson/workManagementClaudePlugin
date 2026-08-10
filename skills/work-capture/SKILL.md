---
name: work-capture
description: >
  Quickly capture a bug, feature idea, improvement, annoyance, tech-debt item, or security
  concern into the project's requests.md intake file, without starting a planning session.
  Use when someone says "note this down", "log this", "capture this", "add this to the
  backlog", "remember this for later", "I noticed a bug", "I had an idea", "can you add a
  request for", "flag this for later", or otherwise mentions something in passing that they
  want recorded rather than fixed now. Also use proactively at the end of a session when a
  real defect or improvement was noticed but deliberately left unfixed. Works on any project
  set up with the work-management model; discovers the project from its manifest rather than
  assuming a location. Do NOT use for refining requests, proposing release scope, or closing
  out a release — those are the /work-plan command.
---

# Work capture

Append a single request to the project's intake file and stop. Capture is deliberately
cheap: the point is to record the thought before it is lost, not to refine it.

## Resolve configuration

Follow `${CLAUDE_PLUGIN_ROOT}/skills/work-model/references/config-resolution.md` to locate
`<paths.work>/project.yml`.

If no manifest is found, do not create one and do not write a request anywhere. Say that
this project is not set up for work capture and that `/work-init` will set it up. Offer to
hold the note in the conversation meanwhile.

**Stale schema — proceed anyway, with one line of warning.** This skill has the only exemption
from the compatibility gate. If `model_version` is older than the supported version, capture
still runs, provided the fields it reads are present and well-formed: `paths.work`, `ids.*`,
`taxonomy.request_types`, `taxonomy.priorities`. Those have not moved between schema eras.

Mention the stale schema once, in one sentence, and name `/work-init --upgrade`. Then get on
with the capture. The reasoning: refusing here loses the note, and the note is the whole point —
whereas the schema can be fixed at any time. If a field capture actually needs is missing or
malformed, stop like any other command.

## Write the request

Read the tail of `<paths.work>/requests.md` to find the highest existing request ID, then
allocate the next one using `<ids.request_prefix>` and `<ids.pad>`. Do not reuse an ID, even
if a gap exists — gaps usually mean something was pruned, and reusing the number makes old
git history ambiguous.

Append to the `## Inbox / needs refinement` section, in the model's exact format:

```text
### <request_prefix>-NNN
Request ID: <request_prefix>-NNN
Title: <short, specific title>
Type: <one of taxonomy.request_types>
Status: inbox
Priority: <one of taxonomy.priorities>
Summary: <what was noticed or wanted, in enough detail to be actionable later>
Notes: <optional — only if there is something genuinely worth noting>
Source: <where it came from, e.g. "human request (direct)">
```

Plain `Key: value` lines only. No bullets, no nested headings, no YAML. Omit `Notes:`
entirely rather than writing `Notes: none`.

Choose `Type:` and `Priority:` from the manifest's vocabularies only. If the thing does not
fit any listed type, use the closest and say which you chose — do not invent a type, because
that is precisely the drift that makes the vocabulary meaningless.

## Judgement

**Write enough that it survives.** "Fix the modal" is not a capture. Record what was
observed, where, and what was expected instead. The person reading it in three weeks is
usually the same person who wrote it, and they will not remember.

**Do not refine.** No acceptance criteria, no work items, no dependency analysis, no
implementation opinion. `Status: inbox` means unreviewed, and `/work-plan` does the
reviewing. Capturing a half-refined request is worse than a rough one, because it looks
finished.

**Do not fix it.** If the fix is trivially obvious, still capture rather than implement,
unless the person clearly asked for the fix. Silently fixing something that was mentioned in
passing bypasses the release process the model exists to run.

**Check for an obvious duplicate** among the inbox and refined sections only — a quick scan,
not a full audit. If one exists, say so and ask whether to add a note to the existing
request instead of creating a new one.

**Ask at most one question.** Priority is the one thing worth asking about if it is
genuinely unclear; anything else, make a reasonable choice and note it. Capture that
requires an interview defeats the purpose.

## Report back

State the ID assigned, the title, the type, and the priority. One or two lines. Then stop —
do not offer to refine it or propose a release; those are commands the person will run when
they mean to.

## Capturing several at once

When several things are mentioned together, write them as separate requests with sequential
IDs, and list what you created. Do not bundle unrelated observations into one request to
save effort — they will need splitting later, and the split loses detail.
