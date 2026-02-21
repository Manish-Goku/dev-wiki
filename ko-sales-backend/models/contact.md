---
model: Contact
collection: contacts
id_pattern: C-N (e.g. C-1)
status: active
last_updated: 2026-02-21
tags: [core, pipeline, documents]
depends_on: [pii, lead, agent, pipeline-workflow]
referenced_by: [deal, cohort]
---

# Contact

Converted lead. Created when `profile` field is set on a lead.

## Fields

| Field | Type | Notes |
|-------|------|-------|
| contact_id | String | Unique, C-N |
| pii_id | String | Unique, ref to PII |
| profile | String | e.g. farmer, retailer, dealer |
| profile_details | Mixed | |
| farm_details | Mixed | |
| crop | Mixed | |
| status | Enum | active, dormant, lost, order_placed, etc. |
| disposition | Enum | follow_up, prospect, dormant, lead_closed, order_placed, null |
| call_status | Enum | Same as lead |
| agent_category | String | From assigned agent |
| pipeline_id | String | Active CP-N pipeline |
| document_status | `{adhar, pan, pc}` | Booleans, auto-synced from PII |
| prospect_value | Number | |
| fps / rps | Number | |
| prescription_count | Number | |
| notes | Mixed[] | |
| tags | ObjectId[] | |
| updated_data | Mixed[] | Audit trail |

## Indexes

- `pii_id`, `contact_id`, `status`, `agent_category`, `disposition`

## Key Rules

- Ownership via leads (3-tier visibility: admin → floor manager → agent)
- Contact takes priority over lead for pipelines
- document_status auto-synced from PII.typed_documents
- Emits `entity.updated` event on update (for cohort evaluation)
