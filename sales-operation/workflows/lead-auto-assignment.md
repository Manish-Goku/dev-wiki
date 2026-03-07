# Lead Auto-Assignment System

## Architecture
```
MongoDB leads collection
    │ (change stream: insert)
    ▼
DBListener (utilities/db_listener.py)
    │ background thread, started in FastAPI lifespan
    ▼
assign_lead_with_workflow() (utilities/assignment_logic.py)
    │
    ├── Match workflow: source IN lead_source[] AND state IN states[]
    │
    ├── cluster → round-robin among cluster.agent_ids[]
    ├── agent → direct assign
    └── team → round-robin among agents with matching category
    │
    ▼
Update lead.leadOwner.agentId + log to assignment_logs
```

## Change Stream Listener (`utilities/db_listener.py`)

**Class**: `DBListener`
**Watches**: `operationType: "insert"` on `leads` collection

**Skip conditions**:
1. `manual_assignment == True`
2. `leadOwner.agentId` NOT in unassigned IDs (`A0-1126..A0-1130`)
3. `leadId` already processed within 10s debounce

**Constants**:
```python
UNASSIGNED_AGENT_IDS = ["A0-1126", "A0-1127", "A0-1128", "A0-1129", "A0-1130"]
LEAD_PROCESS_TIMEOUT = 10  # seconds
```

**Graceful degradation**: Silently disables if MongoDB doesn't support change streams (standalone/no replica set).

## Workflow Matching (`utilities/assignment_logic.py`)
```python
# 1. Normalize source (leadSubType) and state to lowercase
# 2. Find first active workflow where:
#    lowercase(lead_source[]) contains normalized source  (OR)
#    lowercase(states[]) contains normalized state         (OR)
# 3. Both must match. First active match wins.
```

## Round-Robin Tracker
```python
class RoundRobinTracker:
    agents: List
    current_index: 0  # wraps around, RESETS on server restart
```
**Not persistent** — index resets when server restarts.

## Assignment Logging
Every assignment creates `assignment_logs` entry:
```javascript
{ lead_id, assigned_agent_id, assign_type, lead_source, lead_state,
  workflow_id, workflow_name, assignment_method: "round_robin"|"manual",
  success, message, assigned_at }
```

## Lifecycle (`workers/lifecycle.py`)
```python
AssignmentSystemLifecycle (singleton via get_lifecycle())
  .startup()      # starts system
  .shutdown()     # stops system
  .health_check() # returns True if running
  .get_status()   # { assignment_system_running, status, error_message }
```
Statuses: `"running"`, `"stopped"`, `"failed"`

## Manual Assignment
1. Create batch: `POST /leads-operation/create-batch`
2. Assign: `POST /leads-operation/assign-leads`
3. Lead request flow: `pending → approved → given → fulfilled`

## State Normalization
100+ variations mapped to canonical names (28 states + 8 UTs).

#active #event-driven
