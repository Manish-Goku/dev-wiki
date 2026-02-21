---
model: PipelineWorkflow
collection: pipelines_workflow
id_pattern: PW-N
status: active
last_updated: 2026-02-21
tags: [config, pipeline]
depends_on: [task-workflow]
referenced_by: [pipeline]
---

# Pipeline Workflow

Configuration for pipeline stages per type + agent_category. Unique index: `{pipeline_type, agent_category}`.

## Fields

| Field | Type | Notes |
|-------|------|-------|
| workflow_id | String | PW-N |
| pipeline_type | Enum | lead, contact, deal |
| agent_category | String | |
| stages | Array | `[{name, index, description, timeline, taskflow_id, required_fields[]}]` |
| is_active | Boolean | |

## 8 Seed Workflows

- Lead: Acquisition (5 stages), Retention (4), Support (3), Company B2B (6)
- Contact: Acquisition (5), Retention (4), Company B2B (5)
- Deal: Acquisition (5: open_deal → quotation_sent → negotiation → closed_won → closed_lost)
