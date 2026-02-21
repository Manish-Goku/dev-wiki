---
model: TaskWorkflow
collection: task_workflows
id_pattern: TW-N
status: active
last_updated: 2026-02-21
tags: [config, automation]
depends_on: []
referenced_by: [task, pipeline-workflow]
---

# Task Workflow

Step definitions for automated task creation per pipeline stage.

## Fields

| Field | Type |
|-------|------|
| task_workflow_id | String (TW-N) |
| name | String |
| related_to_module | String (lead/contact/deal) |
| agent_category | String |
| stage_name | String |
| steps | `[{task_name, description, step_index, priority, sla, due_offset_days}]` |

## 11 Seed Workflows

Across lead (new, contacted, qualified, negotiation stages), contact (onboarding, active), deal (open_deal, quotation_sent, negotiation).
