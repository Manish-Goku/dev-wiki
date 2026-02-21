---
model: DraftOrder
collection: draft_orders
id_pattern: DFT-N
status: active
last_updated: 2026-02-21
tags: [orders, snapshot]
depends_on: [agent, deal, contact]
referenced_by: []
---

# Draft Order

Saved order form snapshots. Converted when order is placed.

## Fields

| Field | Type |
|-------|------|
| draft_order_id | String (DFT-N) |
| agent_id | String |
| deal_id / contact_id | String |
| customer_name / contact_number | String |
| status | Enum: draft, converted |
| order_id | String (set on convert) |
| form_data | Mixed (full OrderFormData snapshot) |
| entry_path | Enum: deal, customer |
| grand_total | Number |

## Key Rules

- Update/delete only if status=draft
- Convert: sets status=converted + order_id
