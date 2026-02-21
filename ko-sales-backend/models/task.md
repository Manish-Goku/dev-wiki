---
model: Task
collection: tasks
id_pattern: T-N (e.g. T-1)
status: active
last_updated: 2026-02-21
tags: [core, sla, automation, supabase]
depends_on: [task-workflow, pipeline, agent, note]
referenced_by: [dashboard]
---

# Task

Work items linked to pipeline stages. Auto-created, SLA-tracked, dual-stored (MongoDB + Supabase).

## Fields

| Field | Type | Notes |
|-------|------|-------|
| task_id | String | T-N |
| task_name / description | String | |
| related_to_module | String | lead, contact, deal |
| related_to_id | String | Entity ID |
| agent_category / agent_id | String | |
| task_type | Enum | pipeline, manual, follow_up, other |
| priority | Enum | low, medium, high, urgent |
| status | Enum | pending, in_progress, completed, overdue, cancelled |
| due_date | Date | |
| sla | String | e.g. "2h", "1d" |
| sla_deadline | Date | Calculated from business hours |
| sla_started_at | Date | |
| sla_status | Enum | on_track, at_risk, breached, completed, not_applicable |
| sla_breach_duration | Number | Minutes breached |
| workflow_id / pipeline_id / step_index | String/Number | Pipeline task linking |
| note_ids | String[] | Bidirectional link to notes |
| completed_by / completed_at | String / Date | |
| pii_id / assigned_by | String | |
