---
model: AgentFreebieQuota
collection: agent_freebie_quotas
source: models/agentFreebieQuotaModel.js
status: active
tags: [#active, #auth-required]
---

# AgentFreebieQuota

Manages per-agent distribution limits for each freebie item. Tracks how many units an agent can distribute within a time period.

## Schema Fields

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `agent_id` | String | Yes | — | Agent identifier (indexed) |
| `freebie_sku` | String | Yes | — | Immutable, uppercase (indexed) |
| `freebie_ref` | ObjectId | Yes | — | Ref to `promotional_items` |
| `quota_limit` | Number | Yes | — | Max distribution per period (min: 0) |
| `quota_used` | Number | No | `0` | Already distributed count |
| `quota_remaining` | Number | — | Calculated | `quota_limit - quota_used` |
| `period` | String (enum) | No | — | `monthly`, `quarterly`, `yearly`, `lifetime` |
| `period_start` | Date | Yes | — | Period begin date |
| `period_end` | Date | No | — | Period end date (null for lifetime) |
| `is_active` | Boolean | No | `true` | Enable/disable quota |
| `usage_history` | Array | No | `[]` | Audit trail |

### usage_history entry

| Field | Type | Notes |
|-------|------|-------|
| `order_id` | String | Associated order |
| `customer_number` | String | Customer phone |
| `quantity` | Number | Units consumed |
| `used_at` | Date | Timestamp |

## Indexes

| Fields | Type | Notes |
|--------|------|-------|
| `agent_id: 1, freebie_sku: 1` | Compound (unique) | One quota per agent per SKU |
| `agent_id: 1, is_active: 1` | Compound | Active quotas per agent |

## Auto-Reset Behavior

When `check_agent_quota()` is called:
1. If `period_end` has passed → reset `quota_used = 0`
2. Update `period_start` to current time
3. Recalculate `period_end` based on period type
4. Lifetime quotas never reset (`period_end = null`)

## Relationships

- `freebie_ref` → `promotional_items._id`
