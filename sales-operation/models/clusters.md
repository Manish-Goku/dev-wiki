# clusters

**Collection**: `clusters` (in CRM-Database)
**Used by**: `leads_operation/`, `utilities/assignment_logic.py`

## Schema
```javascript
{
  _id: ObjectId,
  cluster_id: String,
  name: String,
  description: String,
  agent_ids: [String],                 // agentId references (min 1)
  languages: [String],
  states: [String],
  status: String,                      // "active" | "inactive"
  created_at: DateTime,
  total_leads_assigned: Number,
  last_assigned: DateTime
}
```

## API
- `POST /leads-operation/create-cluster` — `{ name(1-200), description(max 500), agent_ids(unique), languages, states, status }`
- `GET /leads-operation/clusters-list` — list all

## Round-Robin
`RoundRobinTracker` cycles through `agent_ids[]`. Index resets on server restart (not persistent).

#active
