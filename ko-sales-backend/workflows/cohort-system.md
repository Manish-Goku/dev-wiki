---
workflow: Cohort System
status: active
last_worked: 2026-03-05
tags: [#active, #event-driven, #rule-engine, #bullmq]
models: [cohort, cohort-assignment-job, lead, contact, customer, agent]
modules: [cohorts, leads, contacts, customers, agents]
created: 2026-02-21
---

# Cohort System — In-Depth Technical Reference

Rule-based entity segmentation engine with dual evaluation paths (MongoDB aggregation / json-rules-engine) and bulk agent assignment via BullMQ.

## Module Location

`src/modules/cohorts/` — 15 files across schemas, DTOs, services, rule-engine, and processors.

---

## 1. Data Model

### cohorts collection

```typescript
{
  cohort_id: string;       // COH-N (sequential via CounterService)
  name: string;            // required
  description: string;     // optional
  type: 'static' | 'dynamic';
  rules: Record<string, any>;  // json-rules-engine format
  entity_scope: ('lead' | 'contact' | 'customer')[];  // auto-detected
  members: string[];       // array of pii_ids
  member_count: number;
  rule_fields: string[];   // extracted fact names (for dynamic cohort lookup)
  created_by: string;
  last_evaluated_at: Date;
  is_active: boolean;      // default true
  createdAt: Date;         // mongoose timestamps
  updatedAt: Date;
}
// Indexes: { type: 1, is_active: 1 }, { rule_fields: 1 }, { entity_scope: 1 }
```

### cohort_assignment_jobs collection

```typescript
{
  job_id: string;          // CAJ-N (sequential)
  cohort_id: string;
  agent_ids: string[];
  status: 'queued' | 'processing' | 'completed' | 'failed';
  total_leads: number;
  assigned_count: number;
  failed_count: number;
  progress_percent: number; // 0-100
  failures: Array<{ lead_id: string; pii_id: string; error: string }>;
  error_message: string;   // top-level error if job crashed
  initiated_by: string;
  started_at: Date;
  completed_at: Date;
}
// Indexes: { cohort_id: 1 }, { status: 1 }
```

---

## 2. Rule Engine

### 2.1 Field Registry (`rule-engine/field-registry.ts`)

40+ fields mapped across 3 entity types. Each entry:
```typescript
interface field_definition {
  source: 'lead' | 'contact' | 'customer';
  mongo_path: string;  // actual MongoDB field path
  type: 'string' | 'number' | 'date' | 'boolean' | 'array';
}
```

**Lead fields (19):** status, source, state, pin_code, disposition, disposition_reason, call_status, lqs, rating, call_count.answered, call_count.missed_call, call_count.not_connected, lead_owner.agent_id, first_name, last_name, follow_up_date, assigned_date, created_at

**Contact fields (14):** status, profile, disposition, call_status, agent_category, prospect_value, fps, rps, prescription_count, document_status.adhar, document_status.pan, document_status.pc, follow_up_date, created_at

**Customer fields (11):** status, category, customer_profile, customer_value_tier, sector, subsector, pincode, total_order_value, max_order_value, order_count, last_order_date, created_at (mapped to `createdAt`)

**Helper functions:**
- `extract_fields_from_rules(rules)` — walks rule tree, collects unique fact names
- `resolve_entity_scope(fields)` — determines collection scope from field sources
- `get_sources_from_fields(fields)` — returns Set of entity_source

### 2.2 Rule Compiler (`rule-engine/rule-compiler.service.ts`)

Converts json-rules-engine format to MongoDB queries.

**Operator mapping:**
```
equal → direct value
notEqual → { $ne }
lessThan → { $lt }
lessThanInclusive → { $lte }
greaterThan → { $gt }
greaterThanInclusive → { $gte }
in → { $in }
notIn → { $nin }
contains → { $in: [v] }     (for array fields)
doesNotContain → { $nin: [v] }
exists → { $exists }
regex → { $regex, $options: 'i' }
```

**Unsupported operators** (forces json-rules-engine fallback): `containsSku`

**Key methods:**
- `can_compile(rules)` — returns boolean, checks all operators are compilable
- `compile(rules)` — returns `Map<entity_source, filter>` (split by source) — NOTE: `split_by_source()` is incomplete, returns empty map
- `compile_flat(rules)` — returns single flat filter object
- `validate_rules(rules)` — throws BadRequestException on invalid structure/facts/operators

**Value casting:** Numbers, Dates, Booleans auto-cast based on field type definition.

### 2.3 Rule Evaluator (`rule-engine/rule-evaluator.service.ts`)

Dual-path evaluation engine.

**Fast path — MongoDB aggregation (`evaluate_via_mongo`):**
```
lead_model.aggregate([
  // $lookup contacts by pii_id → _contact (if needed)
  // $lookup customers by pii_id → _customer (if needed)
  // $match: compiled rules using _contact./ _customer. prefixes
  // $project: { pii_id: 1 }
]).allowDiskUse(true)
```

The `build_aggregation_match()` method recursively translates rule tree nodes:
- `all` → `$and`
- `any` → `$or`
- leaf → `{ prefix.mongo_path: operator_value }`

Where prefix is: `''` for lead, `'_contact.'` for contact, `'_customer.'` for customer.

**Slow path — json-rules-engine (`evaluate_via_engine`):**
- Batches of 500 leads
- For each batch: hydrate entities → build facts object from FIELD_REGISTRY → run engine
- Collects matching pii_ids

**Single entity evaluation (`evaluate_single`):**
- Hydrates one entity by pii_id
- If compilable → in-memory `match_hydrated_via_mongo()` using recursive tree walk
- If not → json-rules-engine single run

**Entity hydration:**
- `hydrate_entity(pii_id, scope)` — parallel fetch lead + contact? + customer?
- `hydrate_batch(pii_ids, scope)` — bulk fetch with Map-based joining

---

## 3. Cohort CRUD (`cohorts.service.ts`)

### Create
1. `validate_rules()` — structural + field registry validation
2. `extract_fields_from_rules()` → `rule_fields`
3. `resolve_entity_scope()` → `entity_scope`
4. Generate `COH-N` via CounterService
5. Insert document
6. `evaluate_cohort()` — run rules → populate members

### Update
- If `rules` changed → re-validate, re-extract fields, re-detect scope, re-evaluate
- Partial update for name, description, is_active

### Evaluate
```typescript
async evaluate_cohort(cohort):
  pii_ids = await rule_evaluator.evaluate_rules(cohort.rules, cohort.entity_scope)
  updateOne({ cohort_id }, { $set: { members: pii_ids, member_count, last_evaluated_at } })
```

### Static Progress
For static cohorts only. Iterates all members in batches of 500, evaluates each against current rules:
```
{ total: N, still_matching: M, diverged: N-M, progress_percent: round((N-M)/N * 100) }
```

### Dynamic Cohort Updates
```typescript
async evaluate_entity_against_dynamic_cohorts(pii_id, changed_fields):
  // Find dynamic cohorts where rule_fields intersect changed_fields
  for each cohort:
    matched = evaluate_single(pii_id, cohort.rules, cohort.entity_scope)
    if matched && !is_member → $addToSet members
    if !matched && is_member → $pull members
```

---

## 4. Event-Driven Dynamic Updates (`cohort-evaluator.service.ts`)

```typescript
@OnEvent('entity.updated', { async: true })
async handle_entity_updated(event: EntityUpdatedEvent):
  // event = { pii_id, entity_type: 'lead'|'contact'|'customer', changed_fields: string[] }
  prefixed_fields = changed_fields.map(f => `${entity_type}.${f}`)
  await cohorts_service.evaluate_entity_against_dynamic_cohorts(pii_id, prefixed_fields)
```

This is async (non-blocking) — the HTTP response is sent before cohort evaluation.

**To trigger:** Any service that updates a lead/contact/customer should emit:
```typescript
this.event_emitter.emit('entity.updated', {
  pii_id: 'PII-12345',
  entity_type: 'lead',
  changed_fields: ['status', 'disposition'],
});
```

---

## 5. Assignment System (`services/cohort-assignment.service.ts`)

### Constants
- `SYNC_THRESHOLD = 100` — below this, process synchronously
- `PROGRESS_BATCH = 10` — update job doc every N leads

### Assign Flow

```
assign(cohort_id, agent_ids, initiated_by, strategy):
  1. Fetch cohort → validate exists, has members
  2. validate_agents() → all exist + all active
  3. leads_service.find_by_pii_ids(cohort.members) → get lead_id/pii_id pairs
  4. if leads < 100:
       process_assignments() → return { mode: 'sync', result }
  5. else:
       create CAJ-N job doc (status: queued)
       enqueue BullMQ job 'assign-cohort-leads'
       return { mode: 'async', job_id, total }
```

### process_assignments()

```
build lead-agent map based on strategy:
  round_robin: agent_ids[i % agent_ids.length]
  capacity_weighted: proportional to daily_capacity

for each lead:
  try: leads_service.assign_lead(lead_id, agent_id, assigned_by)
       result.assigned++
  catch: result.failed++, push to failures[]

  if job_id && i % 10 == 0: update job progress

if job_id: mark job completed with final counts
return { total, assigned, failed, failures }
```

### Capacity-Weighted Distribution

```typescript
build_capacity_distribution(agent_ids, total_leads):
  1. Fetch each agent's daily_capacity via agents_service
  2. total_capacity = sum(capacities)
  3. For each agent: exact = (capacity / total_capacity) * total_leads
     floored = Math.floor(exact), remainder = exact - floored
  4. leftover = total_leads - sum(floored_counts)
  5. Sort by remainder desc, give 1 extra lead to top agents until leftover = 0

flatten_distribution():
  [{A0-1001, 50}, {A0-1002, 30}] → [A0-1001, ...(50x), A0-1002, ...(30x)]
```

### Agent Validation

```typescript
validate_agents(agent_ids):
  parallel fetch all agents
  check: all exist (throw if any missing)
  check: all is_active (throw if any inactive)
```

---

## 6. BullMQ Processor (`processors/cohort-assignment.processor.ts`)

```typescript
@Processor('cohort-assignment')
class CohortAssignmentProcessor extends WorkerHost:
  async process(job: Job<AssignJobData>):
    set job status → 'processing', started_at
    fetch leads by lead_ids (lean)
    call assignment_service.process_assignments()
    on error: set status → 'failed', error_message
```

**Queue config:**
- Queue name: `cohort-assignment`
- Job name: `assign-cohort-leads`
- `removeOnComplete: { age: 86400 }` (24h)
- `removeOnFail: { age: 604800 }` (7 days)
- `attempts: 1` (no retry)

---

## 7. Controller Endpoints Summary

| Method | Path | Handler |
|--------|------|---------|
| POST | `/cohorts` | create |
| GET | `/cohorts` | get_all (paginated, filterable) |
| GET | `/cohorts/fields` | get_fields |
| GET | `/cohorts/assign-jobs/:job_id` | get_assign_job_status |
| GET | `/cohorts/:cohort_id` | get_by_id |
| GET | `/cohorts/:cohort_id/members` | get_members (paginated) |
| GET | `/cohorts/:cohort_id/progress` | get_progress (static only) |
| POST | `/cohorts/:cohort_id/assign` | assign_leads |
| POST | `/cohorts/:cohort_id/recalculate` | recalculate |
| PUT | `/cohorts/:cohort_id` | update |
| DELETE | `/cohorts/:cohort_id` | delete |

---

## 8. Module Wiring (`cohorts.module.ts`)

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([Cohort, CohortAssignmentJob, Lead, Contact, Customer]),
    BullModule.registerQueue({ name: 'cohort-assignment' }),
    LeadsModule,
    AgentsModule,
  ],
  controllers: [CohortsController],
  providers: [
    CohortsService,
    RuleCompilerService,
    RuleEvaluatorService,
    CohortEvaluatorService,
    CohortAssignmentService,
    CohortAssignmentProcessor,
  ],
  exports: [CohortsService],
})
```

---

## 9. Testing Guide

### Prerequisites
- Redis running on localhost:6379
- MongoDB with KO-Database (leads, contacts, customers, piis, agents collections populated)
- Valid agent IDs in agents collection with `is_active: true`

### Curl Examples

**Create static cohort:**
```bash
curl -X POST localhost:3000/api/v1/cohorts -H 'Content-Type: application/json' \
  -d '{"name":"Test","type":"static","rules":{"all":[{"fact":"lead.status","operator":"equal","value":"New"}]}}'
```

**Create cross-entity dynamic cohort:**
```bash
curl -X POST localhost:3000/api/v1/cohorts -H 'Content-Type: application/json' \
  -d '{"name":"High Value","type":"dynamic","rules":{"all":[{"fact":"customer.total_order_value","operator":"greaterThan","value":10000}]}}'
```

**Assign with round_robin:**
```bash
curl -X POST localhost:3000/api/v1/cohorts/COH-1/assign -H 'Content-Type: application/json' \
  -d '{"agent_ids":["A0-1001","A0-1002"]}'
```

**Assign with capacity_weighted:**
```bash
curl -X POST localhost:3000/api/v1/cohorts/COH-1/assign -H 'Content-Type: application/json' \
  -d '{"agent_ids":["A0-1001","A0-1002"],"strategy":"capacity_weighted"}'
```

**Poll async job:**
```bash
curl localhost:3000/api/v1/cohorts/assign-jobs/CAJ-1 | jq '.job.status, .job.progress_percent'
```

### Verify in MongoDB
```javascript
db.cohorts.find({ cohort_id: "COH-1" }, { member_count: 1, members: { $slice: 5 } })
db.cohort_assignment_jobs.find({ job_id: "CAJ-1" })
```

---

## 10. Bugs Found & Fixed (2026-03-05)

### Fixed
1. **Operator validation missing** — `validate_node()` in `rule-compiler.service.ts` checked fact existence and value presence, but never validated the operator name. Invalid operators (e.g. `"foobar"`) passed validation and caused the server to hang during `build_aggregation_match()`. **Fix:** Added `OPERATOR_MAP` + `UNSUPPORTED_OPERATORS` check, returns 400 with valid operator list.

2. **BSON 16MB overflow** — `evaluate_cohort()` stored ALL matching pii_ids in the `members` array. With 789k active leads matching broad rules, the MongoDB document exceeded the 16MB BSON limit, crashing with `RangeError: offset out of range`. **Fix:** Capped stored members at 500k (`MAX_STORED_MEMBERS`). `member_count` still reflects the true total count.

### Not Fixed (unrelated)
3. **ImapFlow socket timeout** — `EmailPollService` cron (every 1min) triggers ImapFlow IMAP connection that times out → unhandled `'error'` event on ImapFlow instance → crashes entire Node.js process. Needs `.on('error')` handler in `imap.service.ts`. Workaround: temporarily disable `CustomerSuccessModule` during testing.

---

## 11. Test Results (2026-03-05)

**Auth required:** All endpoints need Bearer token. Login via `POST /api/v1/auth/login`.
**Active agents for testing:** `A0-1487` (aditi kumari), `A0-1501` (Koushik Panda)

### Passing (28/30)
| Category | Tests | Status |
|----------|-------|--------|
| Read endpoints | fields, list, search, type filter, get by ID, 404, members pagination | ALL PASS |
| Validation | missing rules, invalid fact/operator/type/value, empty agent_ids | ALL PASS |
| Create | static + dynamic + various operators + auto scope detection | ALL PASS |
| Update | name, is_active toggle, rules change → re-evaluate | ALL PASS |
| Delete | delete + verify 404, delete nonexistent → 404 | ALL PASS |
| Static progress | total=1, still_matching=1, diverged=0, progress_percent=0 | PASS |
| Progress on dynamic | correct 400 rejection | PASS |
| Sync assign | round_robin (1 lead) → assigned=1, failed=0 | PASS |
| Sync assign | capacity_weighted (1 lead) → assigned=1, failed=0 | PASS |
| Async assign | 500k leads → returns job_id=CAJ-2, mode=async | PASS |
| Job polling | returns status=queued, progress_percent, assigned_count | PASS |
| Agent validation | invalid ID → 400, inactive → 400 | PASS |
| Cohort validation | assign nonexistent cohort → 404, nonexistent job → 404 | PASS |

### Performance Notes
- Lead-only aggregation: fast (< 1s for simple $match)
- Cross-entity $lookup: extremely slow on 200k+ leads (minutes+, often times out)
- Recalculate on broad rules: times out (same root cause)
- BullMQ stale jobs can block queue after server crash (need manual Redis cleanup)

### Data Notes (beta DB)
- 789k leads match `status=active`
- 55k leads in MAHARASHTRA alone
- Most leads have `lqs=0`, `rating=0`, empty `call_count`
- Cross-entity requires $lookup into contacts (200k+) and customers — very slow without indexes

---

## 12. Known Issues / Future Work

1. **`get_agent_capacity()`** reads `daily_capacity` from DB directly — needs: `capacity - already_assigned_today`
2. **`split_by_source()`** in rule-compiler returns empty map — aggregation uses `build_aggregation_match()` instead (works but dead code exists)
3. No max-capacity guard on assignment
4. No realtime notification (WebSocket) on async job completion
5. `containsSku` is the only listed unsupported operator — forces slow path
6. Static progress endpoint can be slow for large cohorts (no caching)
7. `entity.updated` events must be emitted by lead/contact/customer services — not all update paths may emit yet
8. Cross-entity $lookup needs compound indexes on `pii_id` in contacts/customers collections
9. BullMQ needs stale job detection/cleanup for crashed worker scenarios
10. Members array cap at 500k — cohorts with more members can't do full assignment (need pagination or external member storage)

---

## 13. Branch History

- **2026-02-21:** Built on `main`
- **2026-03-05:** Merged `main` → `manish-master-v2` (with bug fixes), then `manish-master-v2` → `praveen`
- All three branches (`main`, `manish-master-v2`, `praveen`) now have cohort module + fixes
