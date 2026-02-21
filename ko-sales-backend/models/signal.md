---
model: Signal
collection: signals
id_pattern: SIG-XXXXX
status: active
last_updated: 2026-02-21
tags: [inbound, sla]
depends_on: [pii]
referenced_by: [dashboard]
---

# Signal

Inbound signals from various channels. Used for duplicate lead detection.

## Fields

| Field | Type |
|-------|------|
| signal_id | String (SIG-XXXXX) |
| pii_id | String |
| entity_id | String |
| module | Enum: lead, contact, deal, other |
| signal_type | Enum: whatsapp, call, email, system |
| content | String |
| status | Enum: pending, resolved |
| sla_deadline | Date (default 24h) |
| created_by / resolved_by / resolved_at | String / Date |
