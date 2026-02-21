---
model: CohortAssignmentJob
collection: cohort_assignment_jobs
id_pattern: CAJ-N (e.g. CAJ-1)
status: active
last_updated: 2026-02-21
tags: [assignment, async, bullmq]
depends_on: [cohort, agent, lead]
referenced_by: []
---

# CohortAssignmentJob

Tracks async bulk assignment jobs triggered from cohort → agent assignment.

## Fields

| Field | Type | Notes |
|-------|------|-------|
| job_id | String | Unique, CAJ-N |
| cohort_id | String | Source cohort |
| agent_ids | String[] | Target agents |
| status | String | queued, processing, completed, failed |
| total_leads | Number | Total leads to assign |
| assigned_count | Number | Successfully assigned |
| failed_count | Number | Failed assignments |
| progress_percent | Number | 0-100, updated every 10 leads |
| failures | Mixed[] | `[{lead_id, pii_id, error}]` |
| error_message | String | Top-level error (if job itself fails) |
| initiated_by | String | Agent who triggered |
| started_at | Date | When processing began |
| completed_at | Date | When finished |

## Indexes

- `cohort_id`
- `status`

## API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/cohorts/:cohort_id/assign` | POST | Trigger assignment (body: `{agent_ids}`) |
| `/cohorts/assign-jobs/:job_id` | GET | Poll job status |

## Execution Model

- **< 100 leads:** Sync — processes inline, returns result directly
- **>= 100 leads:** Async — creates job doc + BullMQ job, returns job_id for polling
