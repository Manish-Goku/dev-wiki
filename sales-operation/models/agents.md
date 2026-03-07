# agents

**Collection**: `agents` (in CRM-Database — **shared** with CRM-Backend, Inventory-Management-Backend)
**Used by**: `leads_operation/`, `Reports_and_Analytics/`, `utilities/assignment_logic.py`

## Schema
```javascript
{
  _id: ObjectId,
  agentId: String (unique),            // "A0-1001"
  firstname: String,                   // lowercase 'f'
  lastname: String,
  email: String,
  name: String,                        // sometimes used instead of firstname/lastname
  katyayani_id: String,
  Agent_Category: String,              // CAPITAL A,C — department/team
  category: String,                    // alias for Agent_Category in some contexts
  manager: String,                     // manager's agentId
  manager_id: String,                  // manager ref (used in reports)
  status: String,
  languages: [String],
  slack_id: String,                    // also: slackId in some docs
  active: Boolean,
  phone: String,
  AssignedState: [String],             // CAPITAL A,S
  daily_capacity: Number               // default 50 (from ko-sales-backend)
}
```

## Unassigned Placeholder IDs
`A0-1126, A0-1127, A0-1128, A0-1129, A0-1130` — new/unassigned leads use these.

## Field Name Gotchas
- `Agent_Category` not `agent_category`
- `AssignedState` not `assigned_state`
- `firstname` not `firstName`
- `slack_id` vs `slackId` — both exist depending on data source

## Used In
- Lead assignment (leadOwner.agentId → agents.agentId)
- Reports (manager_id → agentId relationship)
- Order attribution (orders.agent → agentId)
- Call tracking (calls.agentId, calls.katyayani_id)

#active #cross-project
