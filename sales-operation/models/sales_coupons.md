# sales_coupons

**Collection**: `sales_coupons` (in CRM-Database)
**Used by**: `coupons/` module
**Note**: Named `sales_coupons` to avoid clash with existing CRM `coupons` collection (which has `couponId`, `discount`, `couponName` schema from the app).

## Schema
```javascript
{
  _id: ObjectId,
  coupon_code: String,                // unique, auto-gen "CPN-{n}" or manual, stored uppercase
  title: String,                      // display name
  description: String,                // optional

  // Discount
  discount_type: String,              // "percentage" | "fixed"
  discount_value: Number,             // e.g. 20 (20% or ₹20)
  discount_upto: Number | null,       // max cap for percentage type

  // Scope
  scope_type: String,                 // "all" | "product" | "category"
  applicable_products: [String],      // SKU strings (when scope_type = "product")
  applicable_categories: [String],    // category name strings (when scope_type = "category")

  // Flat Filters
  min_order_value: Number,            // default 0
  min_order_quantity: Number,         // default 0
  allowed_payment_modes: [String],    // [] = all allowed
  allowed_states: [String],          // [] = all states
  blocked_states: [String],          // blacklist (takes precedence)
  customer_categories: [String],     // [] = all

  // Nested Conditions (Rule Engine)
  conditions: {                       // null if no advanced rules
    "all": [                          // AND — all must match
      {"name": "order_total", "operator": "greater_than_or_equal_to", "value": 2000},
      {"any": [                       // OR — any must match
        {"name": "customer_state", "operator": "equal_to", "value": "Maharashtra"},
        {"name": "customer_state", "operator": "equal_to", "value": "Karnataka"}
      ]},
      {"name": "product_skus_in_cart", "operator": "shares_at_least_one", "value": ["SKU-001"]}
    ]
  },

  // Limits
  overall_limit: Number,              // total uses allowed (-1 = unlimited)
  per_customer_limit: Number,         // max uses per customer (-1 = unlimited)
  per_agent_limit: Number,            // max agent applications per month (-1 = unlimited)

  // Validity
  valid_from: DateTime | null,
  valid_to: DateTime | null,

  // State
  is_active: Boolean,                 // soft toggle (delete sets false)
  usage_count: Number,                // running total, incremented on apply

  // Metadata
  created_by: String,                 // admin email
  created_at: DateTime,
  updated_at: DateTime,
  version: Number,                    // increments on update
  change_log: [{
    changed_by: String,
    changed_at: DateTime,
    changes: {                        // diff per field
      "field_name": { old: any, new: any }
    }
  }]
}
```

## Indexes (recommended)
- `coupon_code`: unique
- `is_active`: for filtered listing
- `created_at`: for sort

## Related Collection: `sales_coupon_usage`
```javascript
{
  _id: ObjectId,
  coupon_id: String,                  // ref sales_coupons._id (as string)
  coupon_code: String,
  order_id: String,
  customer_id: String,
  agent_email: String,
  discount_applied: Number,           // actual ₹ discount given
  order_value: Number,                // cart value at time
  applied_at: DateTime
}
```

## Rule Engine Operators (12)
| Operator | Description | Types |
|----------|-------------|-------|
| `equal_to` | Exact match | string, number |
| `not_equal_to` | Not equal | string, number |
| `greater_than` | > | number |
| `greater_than_or_equal_to` | >= | number |
| `less_than` | < | number |
| `less_than_or_equal_to` | <= | number |
| `contains` | List contains item | list |
| `not_contains` | List doesn't contain item | list |
| `in` | Item is in list | string |
| `not_in` | Item is not in list | string |
| `shares_at_least_one` | Lists have any overlap | list |
| `contains_all` | List contains all items | list |

## Context Variables (for conditions)
| Variable | Description | Type |
|----------|-------------|------|
| `order_total` | Cart total value | number |
| `order_quantity` | Total items in cart | number |
| `product_skus_in_cart` | SKU strings in cart | list |
| `category_names_in_cart` | Category names in cart | list |
| `customer_state` | Customer's state | string |
| `customer_id` | Customer ID | string |
| `customer_category` | Customer category | string |
| `payment_mode` | Payment mode | string |
| `agent_email` | Agent's email | string |

## Auto-Generated Coupon Codes
Pattern: `CPN-{incrementing_number}` (CPN-1, CPN-2, ...)
Looks for last doc matching `^CPN-\d+$` regex, extracts number, increments.
Manual codes accepted — stored uppercase.

#active
