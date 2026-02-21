---
workflow: Cohort System
status: active
last_worked: 2026-02-21
tags: [#active, #event-driven, #rule-engine, #bullmq]
models: [cohort, cohort-assignment-job, lead, contact, customer, agent]
modules: [cohorts, leads, contacts, customers, agents]
created: 2026-02-21
---

# Cohort System

Rule-based segmentation with hybrid evaluation engine + bulk agent assignment.

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

## Agent Assignment

`POST /cohorts/:cohort_id/assign` → `CohortAssignmentService.assign()`

**Body:** `{ agent_ids: string[], strategy?: "round_robin" | "capacity_weighted" }`

### Strategies
- **`round_robin`** (default) — equal split: `lead[i] → agent[i % count]`
- **`capacity_weighted`** — proportional to `Agent.daily_capacity`: `agent_share = (capacity / total_capacity) * total_leads`

### Flow
```
Validate agent_ids (exist + active)
        │
Fetch cohort → get member pii_ids
        │
Find all leads by pii_ids
        │
    Count leads
    ┌────┴────┐
  <100       ≥100
    │          │
  SYNC      ASYNC (BullMQ)
    │          │
    │       Create CAJ-N job doc
    │       Enqueue to 'cohort-assignment' queue
    │       Return job_id immediately
    │          │
    └────┬─────┘
         │
  Strategy?
  ┌──────┴──────┐
  round_robin   capacity_weighted
  │             │
  i % count     fetch capacities →
                proportional share
         │
  For each lead → leads_service.assign_lead()
    (full cascade: close old pipeline, create new, trigger tasks)
         │
  Track: assigned_count, failed_count, failures[]
  Progress: update job doc every 10 leads (async path)
         │
  Return result / mark job completed
```

### Capacity Distribution Example
```
100 leads → 3 agents (capacity: 50, 30, 20)
Total capacity = 100
Agent A0-1001: 50 leads (50%)
Agent A0-1002: 30 leads (30%)
Agent A0-1003: 20 leads (20%)
→ All agents at equal utilization
```

### Infrastructure
- **Queue:** BullMQ (`@nestjs/bullmq`) → Redis
- **Processor:** `CohortAssignmentProcessor` (in-process NestJS worker)
- **Job tracking:** MongoDB `cohort_assignment_jobs` collection (persistent, queryable)
- **Capacity source:** `Agent.daily_capacity` field via `AgentsService.get_agent_capacity()`

### Key Files
- `src/modules/cohorts/services/cohort-assignment.service.ts` — core logic
- `src/modules/cohorts/processors/cohort-assignment.processor.ts` — BullMQ worker
- `src/modules/cohorts/schemas/cohort-assignment-job.schema.ts` — job schema
- `src/modules/agents/agents.service.ts` — `get_agent_capacity()`
