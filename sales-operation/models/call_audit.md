# call_audit

**Collection**: `call_audit` (in CRM-Database)
**Used by**: `agronomy/` module

## Schema
```javascript
{
  _id: ObjectId,
  call_id: String,                          // references calls.callId

  // Dynamic scoring parameters (from skills_library collection)
  // Each skill is stored as a top-level field using its `code` (e.g., "call_opening", "product_clarity")
  // Score range: 0 to skill's max_score (typically 10)
  // Example built-in skills:
  call_opening: Number,                     // max 10
  call_closing: Number,                     // max 10
  product_clarity: Number,                  // max 10
  farmer_engagement: Number,                // max 10
  solution_relevance: Number,               // max 10
  objection_handling: Number,               // max 10
  urgency_creation: Number,                 // max 10
  buying_signal_response: Number,           // max 10
  product_recommendation: Number,           // max 10
  follow_up_plan: Number,                   // max 10
  overall_communication: Number,            // max 10
  information_gathering: Number,            // max 10
  call_confidence: Number,                  // max 10
  farmer_satisfaction: Number,              // max 10
  // Custom skills added via Skills Library also appear as top-level fields:
  // testing_skill: Number, nothing: Number, etc.

  // computed
  total_score: Number,                      // sum of all active skill scores
  max_total_score: Number,                  // sum of all active skills' max_score values
  percentage_score: Number,                 // (total_score / max_total_score) * 100

  // denormalized fields (for fast skill matrix aggregation)
  agent_email: String,                      // from calls.agent_id (agent's email)
  agent_category: String,                   // from agents.Agent_Category via calls

  // metadata
  auditor_id: String,                       // auditor's email
  auditor_name: String,                     // auditor's display name or email
  remarks: String,
  status: String,                           // "submitted"
  submitted_at: DateTime,
  updated_at: DateTime
}
```

## Dynamic Skills Architecture

Skills are no longer hardcoded. They come from the `skills_library` collection:

```javascript
// skills_library collection
{
  _id: ObjectId,
  code: String,           // e.g., "call_opening", "testing_skill"
  name: String,           // e.g., "Call Opening", "Testing Skill"
  category: String,       // e.g., "Communication"
  description: String,
  importance: String,     // "high", "medium", "low"
  max_score: Number,      // typically 10
  is_system: Boolean,     // true = built-in, can't delete
  is_active: Boolean,     // only active skills are used in audits
  created_at: DateTime,
  updated_at: DateTime
}
```

### How it works:
1. **GET `/agronomy/call-audits/parameters`** returns active skills from `skills_library`
2. Frontend renders scoring inputs dynamically based on returned parameters
3. **POST `/agronomy/call-audits/submit`** receives scores as `{skill_code: score}` dict
4. Backend `create_call_audit()` reads active skills, stores each score as a top-level field
5. `total_score` = sum of all scores, `max_total_score` = sum of all `max_score` values
6. `percentage_score` = `(total_score / max_total_score) * 100`

### Pydantic Response Model:
`CallAuditItemResponse` uses `model_config = {"extra": "allow"}` so any dynamic skill field passes through without needing to be declared:
```python
class CallAuditItemResponse(BaseModel):
    model_config = {"extra": "allow"}
    audit_id: str
    call_id: str
    total_score: int = 0
    max_total_score: int = 0
    # ... other declared fields
    # Dynamic skill fields (call_opening, testing_skill, etc.) pass through via extra
```

**Important**: `_id` (bson.ObjectId) must be popped from audit dicts before Pydantic serialization — `audit.pop("_id")` not `audit["_id"]` — otherwise `extra: "allow"` causes Pydantic to try serializing ObjectId and fail with 500.

### CallDetails Model:
```python
class CallDetails(BaseModel):
    lead_name: Optional[str] = None
    agent_name: Optional[str] = None
    agent_category: Optional[str] = None
    call_date: Optional[datetime] = None
    duration: Optional[int] = None
    call_recording: Optional[str] = None    # URL to call recording audio
    phone_number: Optional[str] = None
```

## Aggregations

### Dashboard Stats (GET /call-audits/stats)
Returns counts for the full date range (not paginated):
```python
{
  "total_calls": int,      # total connected calls in date range
  "audited_count": int,    # calls with audit records
  "pending_count": int,    # total_calls - audited_count
  "breach_count": int,     # unaudited calls older than 4 hours
  "avg_score": float       # average total_score across all audits
}
```
- SLA breach = unaudited call with `datetime` > 4 hours ago
- Frontend dashboard cards use this endpoint (accurate across all data, not just current page)

### Skill Matrix (GET /agronomy/skill-matrix)
Uses a **precomputed `skill_matrix` collection** — NOT live aggregation.

**Write**: `refresh_agent_skill_matrix(db, agent_email)` called on every audit create/update. Aggregates that agent's `call_audit` records, computes avg per skill, upserts into `skill_matrix`.

**Read**: Simple `find()` on `skill_matrix` with pagination (`page`/`limit`), department filter, sorted by `audit_count` desc.

**Performance**: ~0.3s response (was 75s with aggregation pipeline due to `$facet` IOPS throttling on shared MongoDB clusters).

**Why not aggregation**: MongoDB `$facet` on shared/beta tier causes ~80s delays even on 14 documents. Individual pipeline stages run in <0.05s each, but `$facet` (or running the full pipeline twice for count+data) triggers severe IOPS throttling. Precomputed collection eliminates all aggregation at read time.

See `skill_matrix` collection schema in `models/skill_matrix.md`.

## Query Filters (audit-list endpoint)
- `call_id` -- filter by specific call
- `auditor_id` / `auditor` -- filter by auditor email
- `start_date` / `end_date` -- date range on `submitted_at` (YYYY-MM-DD)
- `status` -- filter by audit status (e.g., "submitted")
- `department`, `sla_status` -- accepted but not DB-filtered (frontend-only filters)

## Frontend Integration

### Dashboard Cards
- Uses `useCallAuditStats(startDate, endDate)` hook -> GET `/api/agronomy/call-audits/stats`
- Shows: Total Calls, Completed, Pending, SLA Breach, Avg Score
- Accurate counts for entire date range (not just current page of 25 records)
- Auto-refreshes on date change, manual refresh, and audit submission

### Tab Badges
- All tab: `stats.total_calls`
- Pending tab: `stats.pending_count`
- Completed tab: `stats.audited_count`
- SLA Breach tab: `stats.breach_count`

### Completed Audits
- Fetched via GET `/api/agronomy/audit-list` with date/filter params
- Response includes `call_details` with `call_recording` URL
- "All" tab merges audit data into call records using `auditDataByCallId` Map keyed on `call_id`
- "View Details" opens read-only dialog with dynamic score breakdown, percentage, remarks, and inline audio player
- "Complete Audit" opens editable form with dynamic scoring fields + remarks + inline audio player + submit

### Score=0 Bug Fix
The dropdown menu uses `call.total_score != null` (not truthiness) to decide between "Complete Audit" and "View Details". This prevents audits with score 0 from incorrectly showing "Complete Audit" instead of "View Details".

### Inline Audio Player
Both the "Complete Audit" and "View Details" dialogs embed a native HTML `<audio controls>` player in the call info header when `call_recording` URL is available. This allows auditors to listen to the recording while scoring without opening a separate popup.

### Score Inputs
- `type="text"` + `inputMode="numeric"`, strips non-digits, clamps 0-10

## Audit Status in Call Logs ($lookup)

The connected calls pipeline (`get_connected_calls` in `agronomy/queries.py`) includes a `$lookup` from `call_audit` to return audit status inline:

```javascript
// Added to pipeline via $lookup + $addFields:
{
  is_audited: { $gt: ["$audit_match", []] },   // true if matching audit exists
  total_score: "$audit_match.total_score",       // first match
  max_total_score: "$audit_match.max_total_score",
  percentage_score: "$audit_match.percentage_score",
  audit_id: { $toString: "$audit_match._id" }
}
```

This eliminates the need for frontend to maintain a separate audit lookup map. The `is_audited` flag drives:
- Menu logic: "Complete Audit" vs "View Details"
- Badge display: colored percentage on audited rows

## Agent Detail Dialog Integration

`get_call_audits_by_email(db, agent_email)` in `dashboard/queries.py` returns all audits for an agent with skill breakdowns:

```python
# Response model: dashboard/response_model.py → CallAudit
class CallAudit(BaseModel):
    audit_id: Optional[str]
    call_id: Optional[str]
    audit_date: Optional[datetime]
    audited_by: Optional[str]
    total_score: Optional[int]
    max_total_score: Optional[int]
    percentage_score: Optional[float]
    category: Optional[str]
    skill_scores: Optional[Dict[str, int]]  # {"call_opening": 10, ...}
```

**Important**: `AgentDetailsResponse` in `dashboard/route.py` must explicitly pass `call_audits=result.get("call_audits", [])` to the constructor — Pydantic won't auto-populate it from the dict.

#active
