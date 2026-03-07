# Agronomy Module

Full-stack module for call quality auditing, agronomy testing, crop suggestions, principal certificates, skill matrix, and product catalog.

## Backend Structure

```
agronomy/
  __init__.py
  route.py           # 21 endpoints
  queries.py          # All DB queries, aggregation pipelines
  request_model.py    # Pydantic request models
  response_model.py   # Pydantic response models
```

**Auth**: All endpoints require `require_pbi_or_retention()`.

---

## 1. Call Audits

### Workflow
```
Auditor opens /agronomy/audits
  -> GET /agronomy/call-audits (connected calls, paginated, 25 per page)
  -> GET /agronomy/call-audits/stats (dashboard counts for full date range)
  -> GET /agronomy/call-audits/parameters (dynamic skills from skills_library)
  -> Select call to audit -> "Complete Audit" in 3-dot menu
  -> Inline audio player for listening to recording while scoring
  -> Fill dynamic scoring parameters (from skills_library, each 0-max_score)
  -> POST /agronomy/call-audits/submit
  -> Dashboard stats auto-refresh (refetchStats)
  -> Call moves from Pending to Completed tab
  -> View in "Completed" tab via GET /agronomy/audit-list (supports date/status/auditor filters)
  -> "View Details" in 3-dot menu -> read-only dialog with score breakdown + remarks + inline audio player
  -> "All" tab merges audit data (scores, auditor) into call records
```

### Scoring Parameters (Dynamic from skills_library)

Skills are managed via the Skills Library (CRUD endpoints). Default built-in skills:

| Parameter | Max | System |
|-----------|-----|--------|
| call_opening | 10 | yes |
| call_closing | 10 | yes |
| product_clarity | 10 | yes |
| farmer_engagement | 10 | yes |
| solution_relevance | 10 | yes |
| objection_handling | 10 | yes |
| urgency_creation | 10 | yes |
| buying_signal_response | 10 | yes |
| product_recommendation | 10 | yes |
| follow_up_plan | 10 | yes |
| overall_communication | 10 | yes |
| information_gathering | 10 | yes |
| call_confidence | 10 | yes |
| farmer_satisfaction | 10 | yes |
| *(custom skills added via Skills Library)* | configurable | no |

**Formula**: `percentage_score = (total_score / max_total_score) * 100`
- `max_total_score` = sum of all active skills' `max_score` values (dynamic, not hardcoded 140)

**Frontend input**: `type="text"` + `inputMode="numeric"`, strips non-digits, clamps 0-max_score

### Dynamic Skills Architecture
1. Skills stored in `skills_library` collection (code, name, category, max_score, is_active, is_system)
2. GET `/call-audits/parameters` returns active skills — frontend renders scoring form dynamically
3. Scores submitted as `{skill_code: score}` dict — backend stores each as top-level field in `call_audit`
4. `CallAuditItemResponse` uses `model_config = {"extra": "allow"}` so any skill field passes through
5. **Critical**: `_id` must be `.pop()`ed from audit dicts before Pydantic serialization (ObjectId breaks `extra: allow`)

### Inline Audio Player
Both "Complete Audit" and "View Details" dialogs embed a native HTML `<audio controls>` player in the call info header when `call_recording` URL is available. Auditors can listen while scoring without a separate popup.

### Score=0 Bug Fix
Dropdown menu uses `call.total_score != null` (not truthiness) to decide between "Complete Audit" and "View Details". Prevents audits with total_score=0 from incorrectly showing "Complete Audit".

### Server-Side Search

Dropdown selector with 5 options: All Fields, Agent Name, Lead Name, Lead ID, Call ID.
Input is debounced (500ms) before sending to backend.

#### Backend: `build_search_filter(search, search_field, db)`

| Search Field | Strategy | MongoDB Query |
|-------------|----------|---------------|
| `call_id` | Direct regex on calls | `{"callId": {"$regex": search, "$options": "i"}}` |
| `lead_id` | Lookup in leads collection first | Query `leads.leadId` with regex (normalizes KO-/ko- to K0-), get matching leadIds, then `{"customer_id": {"$in": lead_ids}}` |
| `lead_name` | Lookup in leads collection first | Query `leads.firstName` with regex, get matching leadIds, then `{"customer_id": {"$in": lead_ids}}` |
| `agent` | Lookup in agents collection first | `_search_agents_by_name(db, search)` — handles multi-word queries by splitting and matching each word against `firstname`/`lastname` with `$and`. Returns matching emails, then `{"agent_id": {"$in": emails}}` |
| `all` / None | Combines all of the above | `{"$or": [callId regex, customer_id regex, agent_id regex, + lead/agent lookups]}` |

#### Multi-word Agent Name Search (`_search_agents_by_name`)
```python
def _search_agents_by_name(db, search: str) -> List[str]:
    parts = search.strip().split()
    if len(parts) > 1:
        # "Priti Sasmal" -> $and: [{firstname OR lastname matches "Priti"}, {matches "Sasmal"}]
        conditions = [{"$or": [{"firstname": part_regex}, {"lastname": part_regex}]} for part in parts]
        query = {"$and": conditions}
    else:
        query = {"$or": [{"firstname": regex}, {"lastname": regex}]}
    return db["agents"].distinct("email", query)
```

#### Lead ID Normalization
Users often type `KO-` (letter O) instead of `K0-` (zero). The search normalizes this:
```python
import re
normalized = re.sub(r'(?i)^ko-', 'K0-', search)
```

### Dashboard Stats Endpoint
`GET /agronomy/call-audits/stats?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD`

Returns accurate counts for the FULL date range (not just current page):
```json
{
  "total_calls": 14298,
  "audited_count": 5,
  "pending_count": 14293,
  "breach_count": 14272,
  "avg_score": 42.6
}
```

**Why separate endpoint**: The call-audits endpoint returns only 25 records per page. Dashboard cards (Total, Completed, Pending, SLA Breach, Avg Score) need counts across all data. Previously these were computed from page data (wrong). Now they come from this dedicated lightweight endpoint.

**SLA Breach logic**: Unaudited calls with `datetime` older than 4 hours from now.

### Audit Status in Call Logs (Backend $lookup)

The connected calls pipeline now includes a `$lookup` from `call_audit` collection, returning audit status inline with each call record:

```javascript
// Fields added to each call record via $lookup:
{
  is_audited: Boolean,        // true if call_audit record exists for this callId
  total_score: Number,        // from call_audit.total_score (null if not audited)
  max_total_score: Number,    // from call_audit.max_total_score
  percentage_score: Number,   // from call_audit.percentage_score
  audit_id: String            // ObjectId as string for linking to audit detail
}
```

**Frontend**: Audited calls show a colored badge in the table:
- Green (≥70%): `{percentage_score}%`
- Amber (40-69%): `{percentage_score}%`
- Red (<40%): `{percentage_score}%`
- Uses `is_audited` from backend instead of client-side audit map lookup

### Agent Detail Dialog — Call Audits

The `AgentDetailDialog` audits tab shows real audit data via `get_call_audits_by_email(db, agent_email)`:

```python
# Returns per audit:
{
  "audit_id": str,           # ObjectId as string
  "call_id": str,            # e.g., "CO-1771640"
  "audit_date": datetime,    # from submitted_at
  "audited_by": str,         # auditor_name or auditor_id
  "total_score": int,        # e.g., 170
  "max_total_score": int,    # e.g., 170
  "percentage_score": float, # e.g., 100.0
  "category": str,           # agent_category
  "skill_scores": dict,      # {"call_opening": 10, "product_clarity": 8, ...}
  "remarks": str
}
```

**Frontend**: Each audit is a clickable card. Clicking expands to show skill breakdown grid (2 columns, skill name + score/10). Header shows avg percentage across all audits.

**Route fix**: `AgentDetailsResponse` in `dashboard/route.py` must pass `call_audits=result.get("call_audits", [])` — was previously missing from the constructor.

### Endpoints
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/agronomy/call-audits` | Paginated connected calls with agent & lead details + inline audit status ($lookup). Params: start_date, end_date, page, limit, search, search_field |
| GET | `/agronomy/call-audits/stats` | Dashboard counts: total, audited, pending, breach, avg_score |
| GET | `/agronomy/call-audits/parameters` | Dynamic list of active scoring parameters from skills_library |
| GET | `/agronomy/agent-categories` | Distinct Agent_Category values from agents collection (for department filter) |
| POST | `/agronomy/call-audits/submit` | Create audit with dynamic skill scores |
| GET | `/agronomy/calls/{call_id}/audit` | Get audit for specific call |
| PUT | `/agronomy/call-audits/{audit_id}` | Partial update, recalculates total |
| GET | `/agronomy/audit-list` | Paginated audits list, filters: call_id, auditor_id, start_date, end_date, status, auditor, department, sla_status |
| GET | `/agronomy/call-audits/stats/overview` | total_audits, avg/max/min scores |

### Collections
- **calls** (read): `callId`, `outcome`, `datetime`, `agent_id` (email), `customer_id` (lead ID), `duration`, `callRecording`
- **call_audit** (read/write): `call_id`, dynamic score fields (from skills_library, stored as top-level), `total_score`, `max_total_score`, `percentage_score`, `auditor_id`, `auditor_name`, `status`, `submitted_at`
- **skills_library** (read/write): `code`, `name`, `category`, `max_score`, `is_system`, `is_active`, `importance`
- **agents** (lookup): `email`, `firstname`, `lastname`, `Agent_Category`, `agentId`, `phone`, `manager_id`
- **leads** (lookup): `leadId`, `firstName`

### Data Model Relationships
```
calls.customer_id  ->  leads.leadId      (lead name lookup)
calls.agent_id     ->  agents.email      (agent name lookup)
calls.callId       ->  call_audit.call_id (audit status check)
```

**Data overlap**: ~24K calls, ~10K leads, only ~880 match. Most calls show lead_id but no lead_name.

### Frontend Page: CallAudits.tsx
- **Route**: `/agronomy/audits`
- **Hooks**:
  - `useCallLogs(startDate, endDate, limit, page, search?, searchField?)` → GET `/api/agronomy/call-audits`
  - `useCallAuditStats(startDate, endDate)` → GET `/api/agronomy/call-audits/stats`
- **Default filter**: Last 30 days (applied on mount via `handlePresetFilter('30days')`)
- **Pagination**: 25 records per page, server-side pagination with `$skip`/`$limit`
- **Tabs**: All | Pending | Completed | SLA Breach (badges show stats from backend, not page data)
- **Presets**: Today / 7 days / 30 days
- **Filters**: Department (dynamic from `/agronomy/agent-categories`), Status, Auditor, SLA Status (all client-side on current page data)
- **Search**: Dropdown (All Fields, Agent Name, Lead Name, Lead ID, Call ID) + debounced input (500ms). Server-side via `search`/`search_field` query params.
- **Call Details column**:
  - If lead_name exists: shows lead_name + lead_id below in blue mono
  - If no lead_name but has lead_id: shows lead_id as primary text
  - If neither: shows "N/A"
- **3-dot menu actions**:
  - Pending calls (`total_score == null && !is_audited`): "Complete Audit" → opens scoring form dialog with inline audio player + dynamic params from API + remarks + submit
  - Completed calls (`total_score != null || is_audited`): "View Details" → opens read-only dialog with score breakdown, percentage, remarks, inline audio player
  - Play Recording (if available) — opens separate popup player
- **Complete Audit dialog**: Inline `<audio controls>` in header for listening while scoring. Dynamic scoring fields from `/call-audits/parameters`. Remarks + Submit.
- **View Details dialog**: Fully dynamic — fetches parameter definitions from `/call-audits/parameters`, maps scores dynamically. Inline audio player. Max score shows `auditParameters.length * 10`.
- **All tab**: Merges audit data (scores, auditor, total_score) from completed audits into call records via `auditDataByCallId` Map
- **Pending tab**: Filters out calls that have been audited
- **Score=0 fix**: Uses `call.total_score != null` (not truthiness) for menu logic, so audits with score 0 correctly show "View Details"
- **Score inputs**: `type="text"` + `inputMode="numeric"`, strips non-digits, clamps 0-max_score
- **Audio player**: Inline native `<audio controls>` in audit dialogs + separate popup player for table rows
- **Refresh**: Calls `refetch()` + `refetchStats()` (no page reload)
- **IST timezone**: `formatDateToIST()` appends 'Z' to UTC strings from MongoDB, uses `timeZone: 'Asia/Kolkata'`

---

## 2. Agronomy Tests

### Workflow
```
Admin creates test:
  POST /agronomy/tests/create (questions from question bank, assign agents)
  -> FCM notification sent to agents: POST /agronomy/send-test-fcm

Agent takes test:
  /agronomy/test/:testId or /test/:testId (public)
  -> GET test details from CRM API
  -> Answer questions (MCQ, timer: 60min)
  -> POST /agronomy/tests/{test_id}/submit
  -> Auto-evaluation: exact string match for MCQ

Admin reviews:
  -> GET /agronomy/tests (list all)
  -> View answer sheets, launch results
```

### Endpoints
| Method | Path | Purpose |
|--------|------|---------|
| POST | `/agronomy/tests/create` | Create test from question bank |
| GET | `/agronomy/tests/{test_id}` | Get test details |
| PUT | `/agronomy/tests/{test_id}` | Update test metadata |
| GET | `/agronomy/tests` | List tests with filters (department, crop_focus) |
| GET | `/agronomy/question-bank` | Search questions (difficulty, topic, regex search) |
| POST | `/agronomy/question-bank/add` | Add MCQ question |
| POST | `/agronomy/tests/{test_id}/submit` | Submit answers, auto-evaluate |
| POST | `/agronomy/send-test-fcm` | Push test notification to agents |

### Collections
- **agronomy_tests** (read/write): `test_id`, `test_name`, `total_marks`, `specific_agents`, `questions` (ObjectIds), `given_by`, `pending_agents`, `live_on`, `tags`, `state`, `month`
- **agronomy_questions** (read/write): `question_type`, `question`, `image_urls`, `mcq_options`, `answer`, `difficulty`, `topic`, `marks`
- **agronomy_answer_sheets** (write): `katyayani_id`, `test_id`, `answers[]`, `total_marks_obtained`, `out_of`, `is_checked`, `evaluated_by`

### Auto-Evaluation Logic
- MCQ: exact string match on `answer` field
- `is_checked = True`, `evaluated_by = "SYSTEM"`
- Agent moved from `pending_agents` to `given_by` on submission

### Frontend Pages
**AgronomyTests.tsx** (66KB) - `/agronomy/tests`
- **Hook**: `useAgronomyTests()` (13 methods) -> CRM API `https://beta.api.crm.ko-tech.in`
- **Tabs**: Pending | Active | Completed | Results Management
- **Features**: Create test, assign agents, view answer sheets, grade, launch results

**TakeTest.tsx** (35.5KB) - `/agronomy/test/:testId` (fullscreen)
- **Security**: Fullscreen enforcement, tab switch detection (max 2), 60min timer, context menu disabled, back nav prevented
- **Question types**: MCQ (RadioGroup) | Subjective (Textarea)
- **Auto-submit on timeout**, warnings at 5min & 1min

**TestStartPage.tsx** - `/agronomy/test-start` (token discovery entry point)
**TestView.tsx** - `/agronomy/test-view/:testId` (admin preview)
**PublicTestEntry.tsx** - `/test/:testId` (public access with CRM token)
**PublicTestView.tsx** - `/test-session/:sessionId` (session-based test taking)

---

## 3. Agronomy Suggestions

### Workflow
```
Agent creates suggestion for crop issue:
  POST /agronomy/suggestions (with photos/videos via Firebase Storage)

Expert reviews and provides recommendation:
  GET /agronomy/suggestions (list pending)
  POST /agronomy/suggestions/provide (select products, write description)
  -> Sets is_suggested = True
```

### Endpoints
| Method | Path | Purpose |
|--------|------|---------|
| POST | `/agronomy/suggestions` | Create suggestion (form-data + file uploads) |
| GET | `/agronomy/suggestions` | List suggestions, paginated |
| POST | `/agronomy/suggestions/provide` | Submit product recommendation |

### Crop Enums
- **Stages**: Vegetative, Panicle, Flowering, Fruiting, Mature Harvest
- **Irrigation**: Flood, Drip, Sprinkler, Furrow

### Collections
- **agronomy_suggestions** (read/write): `lead_id`, `agent_id`, `crop_name`, `crop_area`, `crop_stage`, `irrigation_method`, `crop_duration`, `issue`, `previous_used_products`, `photo_urls`, `is_suggested`, `suggestion_description`, `products_sku`, `category`

### File Upload
- Firebase Storage: `suggestions_images/{lead_id}_{file_type}_{timestamp}_{uuid}.{ext}`

### Frontend Page: Suggestions.tsx (35.2KB) - `/agronomy/suggestions`
- **Hook**: `useAgronomySuggestions(page, limit, filters)`
- **Tabs**: Waiting | Suggested | Order Placed
- **Product selection**: Search 200+ products, filter by category, multi-select via checkboxes
- **Detail dialog**: Crop info, photos carousel, product recommendation form

---

## 4. Principal Certificates

### Workflow
```
Agent registers B2B agronomy customer:
  POST /agronomy/principal-certificate (form-data with document uploads to S3)
  -> Validates: contact (10 digits), GST (15 chars), pincode (6 digits)
  -> Calculates customer metrics from orders collection
  -> Uploads docs to S3: agronomy/registrations/{email}/{field}_{timestamp}.{ext}
```

### Endpoint
| Method | Path | Purpose |
|--------|------|---------|
| POST | `/agronomy/principal-certificate` | Register customer with doc uploads |

### File Uploads (S3)
- `gst_certificate`, `pesticide_license`, `fertilizer_license`, `aadhaar_front`, `aadhaar_back`, `pan_card`
- Path: `agronomy/registrations/{customer_email}/{field_name}_{timestamp}.{ext}`

### Collections
- **principal_certificates** (write): Full registration with all document URLs, customer metrics
- **orders** (read): Calculate `total_orders_till_now`, `total_order_value_till_now`, `total_months_from_first_order`

### Frontend Page: PCCertificates.tsx (17.7KB) - `/agronomy/pc`
- **Tabs**: Pending | Waiting for Legal | Issued
- **Table columns**: Shop details, contact, agent, total sales, call count, progress badges, status, certificate info
- **Actions**: Call, Assign PC, View Details, Complete Questionnaire, Download Certificate

---

## 5. Skill Matrix

### Architecture: Precomputed Collection

The skill matrix uses a **precomputed `skill_matrix` collection** instead of live aggregation. This was required because MongoDB `$facet` and complex aggregation pipelines on shared/beta clusters have severe IOPS throttling (~75s for 14 audits).

#### Write Path (on every audit)
When an audit is created or updated, `refresh_agent_skill_matrix(db, agent_email)` is called:
1. Aggregates all `call_audit` records for that agent (`$match` + `$group` with `$avg` per skill)
2. Looks up agent info from `agents` collection (name, category)
3. Computes `overall_score` as percentage: `(sum_of_avg_scores / sum_of_max_scores) * 100`
4. Upserts into `skill_matrix` collection

```python
def refresh_agent_skill_matrix(db, agent_email: str):
    # Aggregate this agent's audits
    pipeline = [
        {"$match": {"agent_email": agent_email}},
        {"$group": {"_id": "$agent_email", "audit_count": {"$sum": 1},
                     "avg_call_opening": {"$avg": ...}, ...}}  # dynamic per active skill
    ]
    result = audits_col.aggregate(pipeline)
    # Lookup agent name/category from agents collection
    # overall_score = (sum of avg scores / sum of max_score values) * 100
    # e.g., if avg scores sum to 120 and max scores sum to 170 → overall = 70.6%
    skill_matrix_col.update_one({"agent_email": agent_email}, {"$set": {...}}, upsert=True)
```

Called from:
- `create_call_audit()` (line ~342 in queries.py)
- `update_call_audit()` (line ~485 in queries.py)

#### Read Path (API endpoint)
Simple `find()` with pagination — no aggregation needed:
```python
def get_skill_matrix(db, email=None, department=None, page=1, limit=25):
    query = {}
    if email: query["agent_email"] = {"$regex": ...}
    if department: query["agent_category"] = department
    total = skill_matrix_col.count_documents(query)
    results = skill_matrix_col.find(query, {"_id": 0}).sort("audit_count", -1).skip(...).limit(...)
    return {"success": True, "total_agents": total, "page": page, "agents": results, ...}
```

**Performance**: ~0.3s (was 75s with aggregation pipeline).

#### `skill_matrix` Collection Schema
```javascript
{
  agent_email: String,       // unique, indexed
  agent_name: String,        // "Priti Sasmal"
  agent_category: String,    // "Retention", indexed
  overall_score: Number,     // avg of all skill averages
  audit_count: Number,       // total audits for this agent, indexed
  skills: [                  // precomputed skill breakdown
    { skill: "Call Opening", code: "call_opening", score: 7.5 },
    { skill: "Product Clarity", code: "product_clarity", score: 6.2 },
    // ... one per active skill
  ],
  updated_at: DateTime
}
```

**Indexes**: `agent_email` (unique), `agent_category`, `audit_count`

#### Initial Population
One-time script `populate_skill_matrix.py` seeds the collection from existing `call_audit` data. Run once after deployment.

### Denormalization in call_audit
`call_audit` documents store `agent_email` and `agent_category` directly (denormalized from `calls` + `agents` at audit creation time). This avoids expensive `$lookup` in the refresh aggregation.

### Endpoints
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/agronomy/skill-matrix` | Precomputed agent skill matrix. Params: `email`, `department`, `page`, `limit` |

### Skills Library Endpoints
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/agronomy/skills` | List all skills from `skills_library` collection |
| POST | `/agronomy/skills` | Create new skill |
| PUT | `/agronomy/skills/{skill_id}` | Update skill |
| DELETE | `/agronomy/skills/{skill_id}` | Delete skill |

### Tracked Skills (Dynamic)
Default system skills: call_opening, call_closing, product_clarity, farmer_engagement, solution_relevance, objection_handling, urgency_creation, buying_signal_response, product_recommendation, follow_up_plan, overall_communication, information_gathering, call_confidence, farmer_satisfaction

Custom skills can be added via Skills Library CRUD. All active skills from `skills_library` are included in audit forms, scoring, and skill matrix aggregation + precomputed collection.

### Frontend Page: SkillMatrix.tsx - `/training/skills`
- **Hook**: `useSkillMatrix(page, limit, department)` (with `useCallback` + exposed `refetch`) → GET `/api/agronomy/skill-matrix`
- **Tabs**: Agent Skills | Trainings | Skills Library
- **Grade system**: Score 7+ = A (green), 4-6 = B (orange), <4 = C (red)
- **Stats**: Total Skills, Agents Tracked, Avg Score, Active Training
- **Filters**: Skill, Level (A/B/C), Department (dynamic from `/agronomy/agent-categories`)
- **Pagination**: Server-side, Prev/Next buttons, page X of Y display
- **Department filter**: Server-side via `department` query param (same options as Call Audits). Changing department resets page to 1.
- **Overall score display**: Percentage-based (e.g., "70.6%") — matches backend formula `(sum_avgs / sum_max_scores) * 100`
- **Create Training dialog** (UI only, no backend persistence yet)
- **SPA navigation fix**: `useSkillMatrix()` initializes `loading: true` and uses `useCallback` to ensure data loads on SPA navigation (not just on full page reload)
- **Agent detail integration**: Clicking an agent row opens `AgentDetailDialog` with `agentEmail` prop, which fetches audits from `GET /dashboard/agent-details?agent_email=` including `call_audits` array with skill breakdowns

---

## 6. Products

### Endpoint
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/agronomy/products` | Active products with SKU, display name, category |

### Collections
- **products** (read): `sku`, `displayName`, `isActiveWebListing`, `category` (ObjectId)
- **categories** (lookup): `name`

---

## External Integrations
| Service | Used For |
|---------|----------|
| Firebase Storage | Suggestion photo/video uploads |
| AWS S3 | Principal certificate document uploads |
| FCM | Test notification push to agents |
| CRM API (`beta.api.crm.ko-tech.in`) | Tests CRUD, question bank, answer sheets (frontend direct) |

---

## All Frontend Routes
| Route | Page | Layout |
|-------|------|--------|
| `/agronomy/audits` | CallAudits | Sidebar |
| `/agronomy/tests` | AgronomyTests | Sidebar |
| `/agronomy/test-start` | TestStartPage | Sidebar |
| `/agronomy/test-view/:testId` | TestView | Sidebar |
| `/agronomy/test/:testId` | TakeTest | Fullscreen |
| `/test/:testId/*` | TakeTest | Fullscreen |
| `/test-session/:sessionId` | TakeTest | Fullscreen |
| `/agronomy/pc` | PCCertificates | Sidebar |
| `/agronomy/suggestions` | Suggestions | Sidebar |
| `/training/skills` | SkillMatrix | Sidebar |

## All Hooks
| Hook | API | Source |
|------|-----|--------|
| `useCallLogs(startDate, endDate, limit, page, search?, searchField?)` | GET `/api/agronomy/call-audits` | Local backend |
| `useCallAuditStats(startDate, endDate)` | GET `/api/agronomy/call-audits/stats` | Local backend |
| `useAgronomyTests()` | CRM API (13 methods) | `beta.api.crm.ko-tech.in` |
| `useAgronomyApi()` | Generic wrapper | Local backend |
| `useAgronomySuggestions(page, limit, filters)` | GET `/agronomy/suggestions` | Local backend |
| `useSkillMatrix(page, limit, department)` | GET `/api/agronomy/skill-matrix` | Local backend |

#active #auth-required
