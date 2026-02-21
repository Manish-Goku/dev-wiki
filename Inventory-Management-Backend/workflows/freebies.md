---
workflow: Freebie System
status: active
last_worked: 2026-02-21
tags: [#active, #completed, #auth-required, #public-endpoint]
models: [FreebieRule, FreebieUsageLog, AgentFreebieQuota, PromotionalItem]
modules: [freebies]
---

# Freebie System

Rule-based promotional freebie engine using `json-rules-engine`. Evaluates customer eligibility based on configurable conditions and manages distribution via agent quotas and customer recurrence limits.

## Architecture

```
routes/freebies/
├── route.js                    ← Auto-loaded by routes/index.js
├── controllers/controller.js   ← 11 API endpoints
├── utils/freebieRuleEngine.js  ← Core evaluation engine
├── validators/                 ← 8 express-validator files
│   ├── checkFreebieEligibilityValidator.js
│   ├── createRuleValidaor.js
│   ├── updateRuleValidator.js
│   ├── createAgentQuotaValidator.js
│   ├── updateAgentQuotaValidator.js
│   ├── recordFreebieUsageValidator.js
│   ├── getCustomerUsageValidator.js
│   └── getAgentUsageValidator.js
└── documentation/              ← 9 markdown docs
```

## API Endpoints

### Eligibility (Public)

```
POST /api/freebies/check-eligibility    #public-endpoint
```

Request:
```json
{
  "customer_number": "9876543210",
  "cart_items": [{ "sku": "PROD-001", "quantity": 2 }],
  "delivery_address": { "state": "MAHARASHTRA", "city": "Mumbai", "pincode": "400001", "zone": "NORTH", "cluster": "MH-NOR" },
  "payment_mode": "prepaid",
  "current_order_value_before_discount": 2500,
  "current_order_value_after_discount": 2300,
  "agent_id": "AGT-001"
}
```

Response:
```json
{
  "success": true,
  "data": {
    "eligible_freebies": [{
      "rule_id": "...", "rule_name": "...", "priority": 100,
      "freebie_skus": ["PROMO-TSHIRT-001"],
      "freebie_details": [{ "_id": "...", "name": "T-Shirt", "sku": "...", "type": "product", "available_quantity": 50 }],
      "message": "Eligible for T-Shirt"
    }],
    "ineligible_freebies": [{
      "rule_id": "...", "rule_name": "...",
      "failed_skus": [{ "freebie_sku": "PROMO-001", "reason": "OUT_OF_STOCK" }],
      "reason_code": "ALL_FREEBIES_FAILED"
    }]
  }
}
```

### Rules CRUD (#auth-required)

```
POST   /api/freebies/rules       ← Create rule
GET    /api/freebies/rules       ← List rules (paginated, ?is_active, ?page, ?limit)
GET    /api/freebies/rules/:id   ← Get rule (populated freebie refs)
PUT    /api/freebies/rules/:id   ← Update rule (versioned if conditions change)
```

### Agent Quota CRUD (#auth-required)

```
POST   /api/freebies/agent-quota       ← Create quota
GET    /api/freebies/agent-quota       ← List quotas (?agent_id, ?freebie_sku, ?is_active)
PUT    /api/freebies/agent-quota/:id   ← Update quota
DELETE /api/freebies/agent-quota/:id   ← Delete quota
```

### Usage Tracking (#auth-required)

```
POST /api/freebies/record-usage       ← Record freebie distribution
GET  /api/freebies/customer-usage     ← Customer usage logs (?customer_number, ?freebie_sku, ?status)
GET  /api/freebies/agent-usage        ← Agent usage logs (?agent_id, ?freebie_sku, ?status)
```

## Core Evaluation Flow

```
POST /check-eligibility
  │
  ├─ build_facts()
  │    ├─ From request: order value, delivery address, payment_mode, agent_id, cart_items
  │    └─ From DB: customer_lifetime_order_value, customer_lifetime_order_count
  │         └─ Aggregation on Order collection (only status=15 / delivered)
  │
  ├─ Load active freebie_rules (sorted by priority DESC)
  │
  ├─ For each rule:
  │    ├─ json-rules-engine evaluates conditions against facts
  │    │    ├─ Supports nested all (AND) / any (OR)
  │    │    ├─ Standard operators: equal, notEqual, greaterThan, lessThan, in, etc.
  │    │    └─ Custom operators: notIn, containsSku
  │    │
  │    └─ If conditions pass → for each freebie_sku in rule:
  │         ├─ check_inventory_availability()
  │         │    ├─ Product: direct available_quantity
  │         │    └─ Combo: min(component_qty / item_qty) across all components
  │         │
  │         ├─ check_freebie_recurrence()
  │         │    └─ Query usage_logs for customer within time window
  │         │       Types: lifetime, days, weekly, monthly, quarterly, yearly
  │         │
  │         └─ check_agent_quota()
  │              ├─ Find agent_freebie_quotas doc
  │              ├─ Auto-reset if period expired
  │              └─ Return remaining quota (or 999999 if no quota set)
  │
  └─ Return { eligible_freebies[], ineligible_freebies[] }
```

## Available Facts

| Fact | Type | Source |
|------|------|--------|
| `current_order_value_before_discount` | Number | Request |
| `current_order_value_after_discount` | Number | Request |
| `delivery_state` | String | Request (uppercase) |
| `delivery_city` | String | Request |
| `delivery_pincode` | String | Request |
| `delivery_zone` | String | Request |
| `delivery_cluster` | String | Request |
| `payment_mode` | String | Request (lowercase) |
| `agent_id` | String | Request |
| `cart_items` | Array | Request (`[{ sku, quantity }]`) |
| `customer_lifetime_order_value` | Number | DB aggregation |
| `customer_lifetime_order_count` | Number | DB aggregation |

## Record Usage Flow

```
POST /record-usage { freebie_sku, customer_number, order_id, agent_id, quantity }
  │
  ├─ Create freebie_usage_log (status: "confirmed")
  │
  ├─ If agent_quota_config.enabled on promotional_item:
  │    ├─ Find or create agent_freebie_quotas doc
  │    ├─ Check: quota_used + quantity <= quota_limit (else 400 error)
  │    ├─ Increment quota_used
  │    └─ Append to usage_history
  │
  └─ Return { usage_log, quota_info }
```

## Rule Versioning

| Trigger | Behavior |
|---------|----------|
| Toggle `is_active` only | Simple update, no versioning |
| Change conditions/logic | Old rule → `is_active: false`, `considered_as_log: true`; New rule → `version + 1`, requires `comment` |

## Ineligibility Reason Codes

| Code | Meaning |
|------|---------|
| `OUT_OF_STOCK` | Zero inventory for freebie item |
| `ALREADY_RECEIVED` | Customer recurrence limit reached |
| `AGENT_QUOTA_EXCEEDED` | Agent distribution limit reached |
| `ALL_FREEBIES_FAILED` | All freebies under the rule failed validation |

## Quota Systems (Dual)

### Agent Quota (Distribution Control)
Stored in `agent_freebie_quotas`. Limits how many units an agent can distribute per period.
- Periods: monthly, quarterly, yearly, lifetime
- Auto-resets when period expires
- No quota doc = unlimited (returns 999999)

### Customer Quota (Recurrence Control)
Stored in `promotional_items.customer_quota_config`. Limits how often a customer can receive a freebie.
- Types: lifetime (once ever), days, weekly, monthly, quarterly, yearly
- Checked via `freebie_usage_logs` query (confirmed/delivered status only)

## Integration with Orders

**Standalone module** — not automatically embedded in order creation flow.

Expected integration pattern:
1. During order → call `check-eligibility`
2. Agent/customer selects freebies
3. Order created
4. Call `record-usage` to log distribution
5. Usage log status updates with order lifecycle

## Example Rule

```json
{
  "rule_name": "Premium Q1 Prepaid Campaign",
  "freebie_skus": ["PROMO-TSHIRT-001"],
  "freebie_refs": ["65a1b2c3d4e5f6..."],
  "priority": 50,
  "conditions": {
    "all": [
      { "fact": "customer_lifetime_order_value", "operator": "greaterThanInclusive", "value": 5000 },
      { "fact": "current_order_value_after_discount", "operator": "greaterThanInclusive", "value": 1000 },
      { "fact": "delivery_state", "operator": "in", "value": ["MAHARASHTRA", "GOA", "KARNATAKA"] },
      { "fact": "payment_mode", "operator": "equal", "value": "prepaid" },
      { "fact": "cart_items", "operator": "containsSku", "value": { "sku": "NEEM-OIL-1L", "min_quantity": 1 } }
    ]
  },
  "event": { "type": "FREEBIE_ELIGIBLE", "params": { "message": "Premium customer reward!" } }
}
```

This rule gives a T-shirt to prepaid customers in MH/GOA/KA who have lifetime value >= 5000, current order >= 1000, and have Neem Oil in their cart.
