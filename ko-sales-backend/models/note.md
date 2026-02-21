---
model: Note
collection: notes
id_pattern: ObjectId
status: active
last_updated: 2026-02-21
tags: [shared, linked]
depends_on: [pii]
referenced_by: [lead, contact, deal, task]
---

# Note

Standalone notes referenced by ObjectId[] on entities.

## Fields

| Field | Type |
|-------|------|
| pii | String |
| author | String |
| agent_category | String[] |
| title / description | String |
| pipeline_id | String |
| type | Enum: lead, deal, contact, other |
| task_id | String (optional, bidirectional link) |
