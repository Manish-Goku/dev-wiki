---
model: LeadPipeline / ContactPipeline / DealPipeline
collection: lead_pipelines / contact_pipelines / deal_pipelines
id_pattern: LP-N / CP-N / DP-N
status: active
last_updated: 2026-02-21
tags: [core, pipeline, task-trigger]
depends_on: [pipeline-workflow, agent]
referenced_by: [lead, contact, deal, task]
---

# Pipeline

Tracks entity progress through workflow stages.

## Fields (shared across LP/CP/DP)

| Field | Type | Notes |
|-------|------|-------|
| pipeline_id | String | LP-N / CP-N / DP-N |
| workflow_id | String | Ref to PW-N |
| pii_id | String | |
| lead_id / contact_id / deal_id | String | Entity ref |
| agent_id | String | |
| agent_category | String | |
| current_stage | String | Stage name |
| stage_history | Array | `[{stage, entered_at, exited_at, moved_by}]` |
| status | Enum | active, closed, completed |
| closed_reason | String | owner_changed, lead_lost, completed, deal_created, null |

## Key Rules

- Forward-only transitions (target.index > current.index)
- Stage transition triggers task creation via taskflow_id
- Old stage pending tasks cancelled on transition
- Reassignment closes pipeline (reason: owner_changed) → creates new one
