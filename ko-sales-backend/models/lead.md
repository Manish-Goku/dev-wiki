---
model: Lead
collection: leads
id_pattern: K0-1XXXXX (e.g. K0-100001)
status: active
last_updated: 2026-02-21
tags: [core, assignment, pipeline]
depends_on: [pii, agent, pipeline-workflow]
referenced_by: [contact, deal, lead-activity, signal, cohort]
---

# Lead

Base sales entity. Every person enters the system as a lead.

## Fields

| Field | Type | Notes |
|-------|------|-------|
| lead_id | String | Unique, uppercase, K0-1XXXXX |
| pii_id | String | Ref to PII (string, NOT ObjectId) |
| lead_owner | `{agent_id, agent_ref}` | Assigned agent |
| first_name | String | Lowercase |
| last_name | String | Lowercase |
| state | String | Uppercase (e.g. UP, MH) |
| pin_code | String | |
| source | String | Default: 'other' |
| status | Enum | active, dormant, contact_created, order_placed, lost, unqualified, not_performed, disable, blacklisted |
| disposition | Enum | follow_up, push_to_advisory, order_placed, lead_closed, prospect, unqualified, reassign, dormant, lost, null |
| disposition_reason | String | |
| call_status | Enum | busy, answered, switched_off, missed_call, rejected, voice_mail, connected, '' |
| call_count | Object | `{answered, missed_call, not_connected, switched_off}` — auto-incremented |
| lqs | Number | Lead Quality Score |
| rating | Number | 0-5 |
| pipeline_id | String | Active LP-N pipeline |
| follow_up_date | Date | |
| assigned_date | Date | |
| assigned_by | String | |
| notes | ObjectId[] | Ref to Notes |
| tags | ObjectId[] | |
| updated_data | Mixed[] | Audit trail entries |

## Indexes

- `pii_id`, `lead_owner.agent_id`, `status`, `disposition`, `source`, `pipeline_id`

## Key Rules

- One lead per phone (duplicate check via PII)
- Profile field on lead update → triggers Contact creation
- Assignment cascades to pipeline close/create
- Emits `entity.updated` event on update (for cohort evaluation)
