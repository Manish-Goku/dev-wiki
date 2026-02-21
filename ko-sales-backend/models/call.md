---
model: Call
collection: calls
id_pattern: CALL-N
status: active
last_updated: 2026-02-21
tags: [migration, crm-sync]
depends_on: [pii]
referenced_by: []
---

# Call

Call logs migrated from CRM-Database.

## Fields

| Field | Type |
|-------|------|
| call_id | String (CALL-N) |
| source_call_id | String (unique, from CRM) |
| agent_email / agent_id | String |
| lead_id / pii_id | String |
| phone_number | String |
| call_type | Enum: inbound, outbound |
| duration | Number |
| call_datetime | Date |
| recording_url | String |
| outcome | Enum: connected, missed_call, rejected, successful, unknown |

## Sync

`POST /calls/sync` — admin only, idempotent via source_call_id unique index. Connects to CRM-Database, filters by allowed agent emails, creates PII records for new phones.
