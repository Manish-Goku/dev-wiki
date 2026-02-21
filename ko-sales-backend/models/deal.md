---
model: Deal
collection: deals
id_pattern: D-N (e.g. D-1)
status: active
last_updated: 2026-02-21
tags: [core, pipeline, quotation]
depends_on: [pii, contact, lead, agent, pipeline-workflow]
referenced_by: [draft-order, cohort]
---

# Deal

Sales opportunity linked to a contact.

## Fields

| Field | Type | Notes |
|-------|------|-------|
| deal_id | String | Unique, D-N |
| pii_id | String | |
| contact_id | String | |
| lead_id | String | |
| agent_category | String | |
| agent_id | String | |
| pipeline_id | String | DP-N |
| status | Enum | open, won, lost, closed, order_created |
| probability | Number | |
| validity_days | Number | |
| estimated_closure_date | Date | |
| actual_closure_date | Date | |
| quotation | Array | Embedded quotation sub-docs (Q-N) |
| notes | ObjectId[] | |
| updated_data | Mixed[] | |

## Embedded: Quotation

| Field | Type |
|-------|------|
| quotation_id | String (Q-N) |
| line_item | Array |
| status | draft / shared / rejected / accepted / expired |
| payment_type | String |
| total_amount / prepaid_amount / cod_amount | Number |
| expiry_date / created_at | Date |
| channel | String |

## Key Rules

- Deal creation auto-closes contact pipeline (reason: deal_created)
- New quotation auto-expires previous draft/shared quotations
- Quotation lifecycle: draft → shared → accepted/rejected/expired
