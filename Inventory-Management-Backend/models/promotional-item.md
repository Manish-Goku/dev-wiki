---
model: PromotionalItem
collection: promotional_items
source: models/promotionalItems.js
status: active
tags: [#active, #auth-required]
---

# PromotionalItem

Represents freebie items that can be distributed to customers. Supports single products and combo bundles.

## Schema Fields

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `sku` | String | Yes | — | Unique, uppercase |
| `name` | String | Yes | — | Display name |
| `type` | String (enum) | No | — | `product` or `combo` |
| `category` | String | Yes | — | Item category |
| `dimensions` | Object | No | — | Physical dimensions |
| `weight` | Number | Yes | — | Physical weight |
| `images` | [String] | No | `[]` | Product image URLs |
| `is_active` | Boolean | No | `true` | Enable/disable item |
| `available_quantity` | Number | No | — | Current inventory stock |
| `items` | Array | No | — | Combo components (combo type only) |
| `agent_quota_config` | Object | No | — | Per-agent distribution limits |
| `customer_quota_config` | Object | No | — | Customer recurrence limits |
| `quota_updationg_logs` | Array | No | — | Historical quota config changes |

### items[] (combo type)

| Field | Type | Notes |
|-------|------|-------|
| `product` | ObjectId | Ref to another item |
| `item_quantity` | Number | Quantity per combo unit |

### agent_quota_config

```json
{
  "enabled": true,
  "type": "monthly",
  "limit": 50,
  "period": 1
}
```
Types: `lifetime`, `days`, `weekly`, `monthly`, `quarterly`, `yearly`

### customer_quota_config

```json
{
  "enabled": true,
  "type": "lifetime",
  "limit": 1,
  "period": 1
}
```
Same type options as agent_quota_config.

## Inventory Calculation

### Product type
Direct `available_quantity` check.

### Combo type
For each component in `items[]`:
- `component_availability = available_quantity / item_quantity`
- Final availability = `Math.min()` across all components

Example: Combo needs 2x SKU-A + 1x SKU-B
- SKU-A stock: 100 → 50 combos
- SKU-B stock: 30 → 30 combos
- **Result: 30 combos available**

## Relationships

- Referenced by `freebie_rules.freebie_refs[]`
- Referenced by `freebie_usage_logs.freebie_ref`
- Referenced by `agent_freebie_quotas.freebie_ref`
- `items[].product` → self-reference or other promotional items
