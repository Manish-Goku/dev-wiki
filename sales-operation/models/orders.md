# orders

**Collection**: `orders` (in CRM-Database — **shared**, written by Inventory-Management-Backend)
**Used by**: `dashboard/` module

## Schema
```javascript
{
  _id: ObjectId,
  order_id: String (unique),
  customer_id: String,
  customer: { first_name, last_name, phone, addresses: [Object] },
  billing_addr: Object,
  shipping_addr: Object,
  items: [{ sku, qty, amount, name }],
  total_amt: Number,                   // NOT total_amount
  order_status: Number,                // integer status code
  order_date: DateTime,
  date: DateTime,                      // used for date range filtering
  created_at: DateTime,
  agent: String,                       // agentId
  katyayani_id: String,
  agent_info: { Agent_Category: String },
  courier_name: String,
  bookingCourier: String,
  courier: [{ name, courier_name }],
  awb_number: String,
  payment_mode: String,                // "COD" | "Prepaid"
  is_mistake: {
    status: Boolean,                   // true = support pending
    chats: String                      // ref to chat doc
  }
}
```

## Status Codes (Complete)
```
 0: failed              1: ready_to_confirm     2: ready_to_invoice    3: ready_to_book
 4: ready_to_pick       5: ready_to_pack        6: ready_to_check      7: ready_to_racked
 8: ready_to_manifest   9: dispatched           10: cancelled          11: rto
12: merged             13: split               14: in transit          15: delivered
16: rto in transit     17: rto received        18: lost               19: destroy
20: not answered X 1   21: not answered X 2    22: not answered X 3   23: not answered X 4
25: out for delivery   26: pending             30: ambiguous          52: ndr
55: picked             -1: creating Order      -5: cancelled
```

## Dashboard Aggregation Groups
- Support pending: `is_mistake.status == true`
- Pre-dispatch: `status == 4`
- NDR: `status == 52`
- RTO: `status in [11, 16, 17]`
- Delivered: `status == 15`
- Revenue: `status in [1-9, 14, 15]`

## SLA
- 4 working hours from creation (10AM-7PM IST)
- Pending detection: `is_mistake.status=true` + dispatch chat with `department: "dispatch"` + `raised_by` non-empty

## Field Gotchas
- `total_amt` NOT `total_amount`
- `date` for range queries, NOT `order_date`
- `agent` is a string agentId, NOT ObjectId

#active #cross-project
