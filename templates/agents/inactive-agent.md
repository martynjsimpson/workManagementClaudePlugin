---
name: {{AGENT_NAME}}
description: NOT ACTIVE on {{PROJECT_NAME}}. This role does not exist on this project. {{INACTIVE_REASON}} Work aimed at this role goes to {{REDIRECT_TO}}.
model: haiku
---
<!-- generated-by: work-management /work-init — edit project.yml and re-run /work-init --repair -->

This role does not exist on **{{PROJECT_NAME}}**.

{{INACTIVE_REASON}}

Work that would otherwise come here goes to **{{REDIRECT_TO}}**.

If you have been spawned, do not attempt the task. Report that this project has no
{{AGENT_NAME}} and name where the work belongs.

---

This file exists deliberately. `{{AGENT_NAME}}` is a role that agents assume by default on
most projects, and without an explicit tombstone, work gets routed to a role that has no
owner — briefs are written for it, handoffs wait on it, and nothing happens. Declaring the
absence is cheaper than debugging the silence.

Declared in `{{WORK_PATH}}/project.yml` under `inactive_agents`.
