---
model: FreebieUsageLog
collection: freebie_usage_logs
source: models/freebieUsageLogModel.js
status: active
tags: [#active, #auth-required]
---

# FreebieUsageLog

Tracks every freebie distribution to customers. Created when `record-usage` is called after order creation.

## Schema Fields

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `customer_number` | String | Yes | — | Phone number (indexed) |
| `customer_ref` | ObjectId | No | — | Ref to `Customer` collection |
| `freebie_sku` | String | Yes | — | Uppercase SKU (indexed) |
| `freebie_ref` | ObjectId | Yes | — | Ref to `promotional_items` |
| `order_id` | String | Yes | — | Associated order ID |
| `quantity` | Number | No | `1` | Units distributed (min: 1) |
| `agent_id` | String | No | — | Sales agent who facilitated |
| `issued_at` | Date | No | `Date.now` | When freebie was given (indexed) |
| `status` | String (enum) | No | `"pending"` | Current state |

## Status Flow

```
pending → confirmed → delivered
              ↓
          cancelled
```

## Indexes

| Fields | Type | Notes |
|--------|------|-------|
| `customer_number: 1` | Single | Customer lookup |
| `freebie_sku: 1` | Single | SKU lookup |
| `issued_at: 1` | Single | Time-based queries |
| `agent_id: 1` | Single | Agent lookup |

## Used By

- **Recurrence check** (`check_freebie_recurrence`): Queries confirmed/delivered logs within time window to determine if customer already received the freebie
- **Customer usage API**: Lists freebie history per customer
- **Agent usage API**: Lists distributions per agent

## Relationships

- `customer_ref` → `customers._id`
- `freebie_ref` → `promotional_items._id`
