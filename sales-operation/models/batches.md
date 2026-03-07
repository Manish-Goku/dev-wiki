# batches + batch_leads

**Collections**: `batches`, `batch_leads` (in CRM-Database)
**Used by**: `leads_operation/`

## batches
```javascript
{
  _id: ObjectId,
  batch_id: String,
  batch_name: String,
  description: String,
  total_leads: Number,
  assigned_leads: Number,
  unique_leads: Boolean,
  status: String,
  created_at: DateTime,
  metrics: {
    assigned_leads: Number,
    attempted_calls: Number,
    connected_calls: Number,
    orders_count: Number,
    total_order_value: Number,
    avg_call_duration: Number,
    pick_rate: Number,                 // orders / calls
    conversion_rate: Number,           // orders / assigned leads
    call_effort: Number,              // calls / orders
    score: Number                      // weighted formula
  }
}
```

## batch_leads (junction)
```javascript
{
  _id: ObjectId,
  batchId: String,                     // → batches.batch_id
  leadId: String,                      // → leads.leadId
  status: String,
  created_at: DateTime
}
```

## API
- `POST /leads-operation/create-batch` — `{ lead_ids, batch_name, description, batch_id? }`
- `POST /leads-operation/assign-leads` — `{ batch_id, agent_assignments: [{agent_id, leads_count}], request_id? }`
- `GET /leads-operation/batch-details` — batch metrics

#active
