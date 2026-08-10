# Requests

Human-facing intake. Capture ideas, bugs, annoyances, improvements, security concerns,
documentation gaps, and technical debt here. Requests may be rough — they do not need to
be implementation-ready.

Request status describes the state of the human-facing ask, not the implementation status
of every derived work item. Detailed delivery state belongs in `backlog.yml` and
`active-release.md`.

- `inbox` — captured, not yet reviewed.
- `needs-refinement` — reviewed, but needs clarification, scoping, splitting, or human
  input before reliable work items exist.
- `refined` — one or more work items exist, but none are currently selected for delivery.
- `in-active-release` — one or more derived work items are selected in `active-release.md`.
- `partially-done` — some of the outcome has shipped, but meaningful scope remains.
- `done` — the human-facing ask is satisfied.
- `blocked` — cannot proceed until a named dependency resolves. Always include a
  `Blocked on:` field. Reviewed at every planning session.
- `deferred` — deliberately parked with no specific dependency. Not surfaced
  automatically; reviewed only via `/work-review-deferred`.
- `rejected` / `duplicate` — whichever best describes what happened to the ask.

Valid `Type:` values: {{REQUEST_TYPES}}
Valid `Priority:` values: {{PRIORITIES}}

Each request is an H3 heading with its ID, followed by plain `Key: value` lines — not
bullets, not a table, not YAML. Do not invent additional fields.

```text
### {{REQUEST_PREFIX}}-001
Request ID: {{REQUEST_PREFIX}}-001
Title: <short title>
Type: <type>
Status: <status>
Priority: <priority>
Summary: <one or more sentences describing the ask>
Notes: <optional>
Work items: <backlog item IDs once they exist>
Source: <where the request came from>
Done in: <version — only when done or partially-done>
Blocked on: <named dependency — only when blocked>
```

---

## Inbox / needs refinement

*(empty)*

---

## Refined requests

*(empty)*

---

## Done

*(empty)*

---

## Deferred / rejected

*(empty)*
