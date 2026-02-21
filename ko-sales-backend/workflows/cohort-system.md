---
workflow: Cohort System
status: wip
last_worked: 2026-02-21
tags: [#wip, #event-driven, #rule-engine, #needs-discussion]
models: [cohort, lead, contact, customer]
modules: [cohorts, leads, contacts, customers]
created: 2026-02-21
---

# Cohort System

Rule-based segmentation with hybrid evaluation engine.

## Create Flow
```
POST /cohorts → validate rules → extract fields
→ Auto-detect entity_scope from field registry
→ Generate COH-N
→ Evaluate immediately:
   ├── Can compile to MongoDB? → aggregation + $lookup (fast, <1s for 200k)
   └── Complex operators? → json-rules-engine batch (500/batch)
→ Store matching pii_ids as members
```

## Entity Scope Auto-Detection
```
Rule uses customer.* field → scope = [lead, contact, customer]
Rule uses contact.* field  → scope = [lead, contact]
Rule uses lead.* only      → scope = [lead]
```

## Dynamic Evaluation (Event-Driven)
```
Lead/Contact/Customer updated → response sent
→ Emit entity.updated {pii_id, entity_type, changed_fields}
→ CohortEvaluatorService @OnEvent (async)
  → Prefix fields with entity_type
  → Find dynamic cohorts where rule_fields overlap changed_fields
  → For each: hydrate entity (lead+contact+customer by pii_id)
  → Match? → $addToSet member : $pull member
```

## MongoDB Aggregation (Fast Path)
```
leads collection
  $lookup → contacts (by pii_id)
  $lookup → customers (by pii_id)
  $match → compiled rules (with _contact./_customer. prefix)
  $project → { pii_id }
```

## Static Cohort Progress
```
GET /cohorts/:id/progress
→ For each member: still matches rules?
→ Returns: { total, still_matching, diverged, progress_percent }
```

## Pending: Agent Assignment
- Distribution: round-robin, weighted, state-based, manual
- Execution: optimized bulk (batch pipeline close/create)
- Status: `#needs-discussion`
