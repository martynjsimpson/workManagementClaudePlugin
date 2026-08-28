# Requests

Human-facing intake — ideas, bugs, annoyances, improvements, security concerns, docs gaps,
tech debt. Requests may be rough; they don't need to be implementation-ready.

Status here tracks the ask, not delivery — detailed progress lives in `backlog.yml` and
`active-release.md`.

| Status | Meaning |
|---|---|
| `inbox` | Captured, not yet reviewed. |
| `needs-refinement` | Needs clarification, scoping, or splitting before work items exist. |
| `refined` | Work items exist; none are yet selected for delivery. |
| `in-active-release` | A derived work item is selected in `active-release.md`. |
| `partially-done` | Some outcome shipped; meaningful scope remains. Reviewed every planning session — file under `## Refined requests`, and record what's left in `Notes:`. |
| `done` | The ask is satisfied. |
| `blocked` | Needs a `Blocked on:` field naming the dependency. Reviewed every planning session. |
| `deferred` | Parked, no specific dependency. Reviewed only via `/work-review-deferred`. |
| `rejected` / `duplicate` | Whichever fits. |

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
Done in: <version, or SPIKE: <ITEM-ID> — only when done or partially-done>
Blocked on: <named dependency — only when blocked>
Parked since: <version or date it first stalled — only while parked>
Reviewed: <count> — last <version>, <why it is still open> (replaced each review, never appended)
```

`Title:` and `Summary:` are written for a person who has not read the code. State the
effect, not the mechanism; put rule ids, paths and symbol names in `Notes:`; and where the
request asks the human a question, end the `Summary:` with it.

Once `Reviewed:` reaches 5, the next planning session stops restating the item and asks for
a decision — do it, reject it, or park it against a named trigger.

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
