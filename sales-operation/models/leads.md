# leads

**Collection**: `leads` (in CRM-Database — **shared**)
**Used by**: `leads_operation/`, `leads_management/`, `utilities/db_listener.py`, `utilities/assignment_logic.py`

## Schema
```javascript
{
  _id: ObjectId,
  leadId: String (unique),
  firstName: String,                   // camelCase
  contact: String,                     // phone number
  source: String,                      // e.g. "Facebook", "Google Ads"
  leadSubType: String,                 // sub-source
  leadProfile: String,
  State: String,                       // CAPITAL S
  city: String,
  district: String,
  quality: Number,                     // 0-100
  leadOwner: {
    agentId: String                    // assigned agent ID or unassigned placeholder
  },
  manual_assignment: Boolean,          // true = skip auto-assign
  created_at: DateTime,
  assigned_Date: DateTime              // CAPITAL D
}
```

## Auto-Assignment Detection
Change stream checks:
1. `manual_assignment != true`
2. `leadOwner.agentId` in unassigned IDs: `A0-1126..A0-1130`
3. Not processed within 10s debounce

## State Normalization
`STATE_VARIATIONS` dict maps 100+ variations → canonical names:
- `"up"`, `"u.p."`, `"uttarpradesh"` → `"Uttar Pradesh"`
- All 28 states + 8 UTs covered

## Field Gotchas
- `State` with CAPITAL S — NOT `state`
- `assigned_Date` with CAPITAL D
- `firstName` camelCase — NOT `first_name`
- `leadOwner.agentId` — nested, NOT top-level

#active #cross-project
