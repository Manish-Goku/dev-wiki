# calls

**Collection**: `calls` (in CRM-Database — **shared**)
**Used by**: `agronomy/`, `Reports_and_Analytics/`, `utilities/agent_metrics_calculation.py`

## Schema (OLD — current production)
```javascript
{
  _id: ObjectId,
  callId: String,                    // e.g. "CO-3555526" — primary identifier
  customer_id: String,               // lead ID e.g. "K0-1636305" — references leads.leadId
  agent_id: String,                  // agent EMAIL (not agentId) — references agents.email
  phoneNumber: String,               // customer phone number
  outcome: String,                   // "Connected", "connected", "Not Connected", etc.
  desposition: String,               // TYPO — NOT "disposition". Values: "resolved", "connected", etc.
  duration: Number,                  // seconds
  datetime: DateTime,                // UTC (stored without timezone info)
  callRecording: String,             // Firebase Storage URL to mp3
  katyayani_id: String               // optional
}
```

## Schema (NEW — target, in knowledge-base, NOT yet migrated)
```javascript
{
  callId: String,
  pii_id: ObjectId,                  // references piis collection
  agent_id: String,                  // agentId (A0-xxx) NOT email
  Agent_Category: String,
  call_type: String,
  call_time: DateTime,               // replaces datetime
  duration: Number,
  outcome: String,
  callRecording: String
}
```
**WARNING**: OLD schema is still in production. Do NOT use new field names (pii_id, call_time, Agent_Category) for queries.

## Key Relationships
```
calls.customer_id  →  leads.leadId     (get lead name via firstName)
calls.agent_id     →  agents.email     (get agent name via firstname+lastname)
calls.callId       →  call_audit.call_id (check if call has been audited)
```

## Connected Call Detection (Agronomy)
```python
{"outcome": {"$in": ["Connected", "connected"]}}
```
Both capitalizations exist in the database — always use `$in`.

## Aggregation Pipeline (Call Audits page)
```python
pipeline = [
    {"$match": {"outcome": {"$in": ["Connected", "connected"]}, **date_filter, **search_filter}},
    {"$lookup": {"from": "agents", "localField": "agent_id", "foreignField": "email", "as": "agent_info"}},
    {"$lookup": {"from": "leads", "localField": "customer_id", "foreignField": "leadId", "as": "lead_info"}},
    {"$unwind": {"path": "$agent_info", "preserveNullAndEmptyArrays": True}},
    {"$unwind": {"path": "$lead_info", "preserveNullAndEmptyArrays": True}},
    {"$sort": {"datetime": -1}},
    {"$skip": skip}, {"$limit": limit},
    {"$project": {
        "call_id": "$callId",
        "lead_id": "$customer_id",
        "agent_name": {"$concat": ["$agent_info.firstname", " ", "$agent_info.lastname"]},
        "agent_email": "$agent_info.email",
        "agent_id": "$agent_info.agentId",
        "agent_category": "$agent_info.Agent_Category",
        "lead_name": {"$ifNull": ["$lead_info.firstName", None]},
        "call_date": "$datetime",
        "status": "$outcome",
        "duration": "$duration",
        "call_recording": "$callRecording",
        "phone_number": "$phoneNumber"
    }}
]
```

## Data Statistics
- ~24,173 unique `customer_id` values in calls
- ~10,003 leads in `leads` collection
- ~880 overlap (calls with matching lead records)
- Most calls display lead_id but have no lead_name (no matching lead)

## Server-Side Search (build_search_filter)
| Search Field | MongoDB Filter | Notes |
|-------------|---------------|-------|
| `call_id` | `{"callId": regex}` | Direct regex on callId |
| `lead_id` | Query `leads.leadId` → `{"customer_id": {"$in": matching_ids}}` | Normalizes KO-/ko- → K0- (zero). Only returns calls with actual leads. |
| `lead_name` | Query `leads.firstName` → `{"customer_id": {"$in": matching_ids}}` | Regex on firstName |
| `agent` | `_search_agents_by_name()` → `{"agent_id": {"$in": matching_emails}}` | Multi-word: splits into parts, matches each against firstname/lastname with `$and` |
| `all` (default) | `{"$or": [callId, customer_id, agent_id, + lead/agent lookups]}` | Combines all above |

## IST Timezone Display
MongoDB stores `datetime` as UTC without timezone suffix. Frontend `formatDateToIST()` appends 'Z' to the string before parsing to ensure correct UTC→IST conversion using `timeZone: 'Asia/Kolkata'`.

## Field Gotchas
- `callId` (camelCase) — NOT `call_id`
- `customer_id` — this is the LEAD ID (K0-xxx), NOT a MongoDB ObjectId
- `agent_id` — this is the agent's EMAIL, NOT their agentId (A0-xxx)
- `desposition` — historical TYPO, cannot change without breaking all projects
- `datetime` — stored as UTC without 'Z' suffix. Must append 'Z' before JS parsing.
- `outcome` — case-inconsistent: "Connected" and "connected" both exist
- `callRecording` (camelCase) — NOT `call_recording`
- `phoneNumber` (camelCase) — NOT `phone_number`

#active #cross-project
