# workflows

**Collection**: `workflows` (in CRM-Database)
**Used by**: `leads_operation/`, `utilities/assignment_logic.py`, `utilities/db_listener.py`

## Schema
```javascript
{
  _id: String,
  workflow_id: String,
  workflow_name: String,
  description: String,
  lead_source: [String],              // match ANY source
  states: [String],                    // match ANY state
  assign_type: String,                 // "cluster" | "agent" | "team"
  cluster_id: String,                  // required if cluster
  agent_id: String,                    // required if agent
  tem: String,                         // team category name, required if team
  is_active: Boolean,
  created_at: DateTime,
  updated_at: DateTime
}
```

## Matching Logic
```
A workflow matches when BOTH:
1. lead.source (lowercase) IN workflow.lead_source (lowercase) — OR logic
2. lead.State (lowercase) IN workflow.states (lowercase) — OR logic

First active match wins (no priority ordering).
```

## Assignment Types
| Type | Targets | Method |
|------|---------|--------|
| `cluster` | `cluster_id` → clusters.agent_ids[] | Round-robin |
| `agent` | `agent_id` directly | Direct assign |
| `team` | `tem` → all agents with matching category | Round-robin |

## API
- `POST /leads-operation/workflows` — create
- `PUT /leads-operation/workflows/{id}` — update
- `GET /leads-operation/workflows` — list all
- `GET /leads-operation/workflows/{id}` — single
- `GET /leads-operation/workflows/data/reference?type=cluster|agent|team` — dropdown data

#active #event-driven
