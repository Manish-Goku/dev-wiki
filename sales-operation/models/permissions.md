# Permission System

**Collections**: `roles`, `modules`, `permissions_matrix` (in CRM-Database)
**Used by**: `auth/` module

## roles
```javascript
{ _id: ObjectId, key: String, name: String }
// e.g. { key: "sales_admin", name: "Sales Admin" }
```

## modules
```javascript
{
  _id: ObjectId,
  key: String,                         // "orders", "leads", etc.
  name: String,
  submodules: [{ key: String, name: String }]
  // e.g. { key: "all_orders", name: "All Orders" }
}
```

## permissions_matrix
```javascript
{
  _id: ObjectId,
  role_key: String,                    // → roles.key
  module_key: String,                  // → modules.key
  role_name: String,
  submodules: [String],                // allowed submodule keys
  updated_at: DateTime
}
```

## Frontend Module→Route Mapping
```
orders           → all_orders, support_pending, pre_dispatch, pre_delivery, ndr, rto, failed_orders
leads            → lead_requests, lead_quality, fulfill_lead_requests
leads_operation  → lead_master, batch_performance, workflow_settings
inventory_pricing → out_of_stock, special_pricing, coupons
agronomy_training → new_joining, agronomy_tests, pc_certificates, suggestions, skill_matrix
reports_analytics → floor_manager_reports, sales_head_reports, agent_activity, training_sops
incentives_wellbeing → coin_master, happiness_index
administration   → user_management, role_management
```

## API
- `GET /auth/module-access` — all role permissions
- `POST /auth/update-module-access` — `{ role_key, true_modules, false_modules }`
- `POST /auth/toggle-module-access` — `{ role_key, module_key, submodules, enable }`
- `POST /auth/batch-toggle-module-access` — `{ role_key, enable: [...], disable: [...] }`

#active #auth-required
