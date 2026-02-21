---
model: FreebieRule
collection: freebie_rules
source: models/freebieRuleModel.js
status: active
tags: [#active, #auth-required]
---

# FreebieRule

Stores eligibility rules that define which customers qualify for promotional freebies. Uses `json-rules-engine` compatible condition format supporting nested AND/OR logic.

## Schema Fields

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `rule_name` | String | Yes | — | Immutable, human-readable identifier |
| `freebie_skus` | [String] | Yes | — | Uppercase SKUs of freebie items |
| `freebie_refs` | [ObjectId] | Yes | — | Refs to `promotional_items` collection |
| `is_active` | Boolean | No | `true` | Enable/disable rule |
| `priority` | Number | No | `1` | Display order (higher = shown first, min: 1) |
| `conditions` | Mixed | Yes | — | json-rules-engine compatible object |
| `event.type` | String | No | `"FREEBIE_ELIGIBLE"` | Event fired on match |
| `event.params` | Mixed | No | — | Custom metadata (analytics/frontend) |
| `version` | Number | No | `1` | Auto-incremented on condition updates |
| `created_by` | String | No | — | Audit: who created |
| `updated_by` | String | No | — | Audit: who last updated |
| `comment` | String | No | — | Required when updating conditions |
| `considered_as_log` | Boolean | No | `false` | Marks old versions (versioning history) |

## Indexes

| Fields | Type | Notes |
|--------|------|-------|
| `freebie_skus: 1` | Single | SKU lookup |
| `is_active: 1, priority: -1` | Compound | Active rules sorted by priority |
| `freebie_refs: 1` | Single | Ref lookup |

## Condition Structure

Root must have exactly one of `all` (AND) or `any` (OR). Can nest indefinitely.

```json
{
  "all": [
    { "fact": "current_order_value_after_discount", "operator": "greaterThanInclusive", "value": 1000 },
    {
      "any": [
        { "fact": "delivery_state", "operator": "in", "value": ["MAHARASHTRA", "GOA"] },
        { "fact": "customer_lifetime_order_count", "operator": "greaterThanInclusive", "value": 5 }
      ]
    },
    { "fact": "payment_mode", "operator": "equal", "value": "prepaid" }
  ]
}
```

## Supported Operators

| Operator | Type | Description |
|----------|------|-------------|
| `equal` / `notEqual` | String/Number | Equality check |
| `greaterThan` / `greaterThanInclusive` | Number | > / >= |
| `lessThan` / `lessThanInclusive` | Number | < / <= |
| `in` / `notIn` | Array | Value in/not in array |
| `contains` / `doesNotContain` | Array | Array contains/doesn't contain value |
| `containsSku` | Custom | Cart contains `{ sku, min_quantity }` |

## Versioning Behavior

On update with condition/logic changes:
1. Old rule: `is_active → false`, `considered_as_log → true`
2. New rule created: `version → old_version + 1`
3. `comment` field required explaining the change

Simple `is_active` toggle does NOT trigger versioning.

## Relationships

- `freebie_refs[]` → `promotional_items._id`
