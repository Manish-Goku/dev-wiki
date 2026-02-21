---
model: Cohort
collection: cohorts
id_pattern: COH-N (e.g. COH-1)
status: active
last_updated: 2026-02-21
tags: [segmentation, rule-engine, event-driven]
depends_on: [lead, contact, customer]
referenced_by: []
created: 2026-02-21
---

# Cohort

Rule-based entity segmentation. Groups leads/contacts/customers by conditions.

## Fields

| Field | Type | Notes |
|-------|------|-------|
| cohort_id | String | Unique, COH-N |
| name | String | |
| description | String | |
| type | Enum | static, dynamic |
| rules | Mixed | json-rules-engine format `{all/any: [{fact, operator, value}]}` |
| entity_scope | String[] | Auto-detected: lead, contact, customer |
| members | String[] | pii_ids |
| member_count | Number | |
| rule_fields | String[] | Extracted from rules for indexing |
| created_by | String | Agent ID |
| last_evaluated_at | Date | |
| is_active | Boolean | |

## Indexes

- `{type, is_active}`, `{rule_fields}`, `{entity_scope}`

## Rule Format

```json
{
  "all": [
    { "fact": "lead.status", "operator": "equal", "value": "active" },
    { "any": [
      { "fact": "customer.total_order_value", "operator": "greaterThan", "value": 10000 },
      { "fact": "contact.prospect_value", "operator": "greaterThan", "value": 0 }
    ]}
  ]
}
```

## Operators

equal, notEqual, lessThan, lessThanInclusive, greaterThan, greaterThanInclusive, in, notIn, contains, doesNotContain, exists, regex

## Key Rules

- Static: frozen snapshot, manual recalculate only, progress tracking (% diverged)
- Dynamic: event-driven — entity update → evaluate single pii_id against matching cohorts
- Hybrid rule engine: MongoDB aggregation (fast) + json-rules-engine (fallback)
- 40+ fields in field registry across lead/contact/customer
- Agent assignment: `#needs-discussion`
