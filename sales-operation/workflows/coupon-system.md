# Coupon System

Full coupon management with nested condition rule engine, eligibility checking, usage tracking, and visual condition builder UI.

## Backend Structure

```
coupons/
  __init__.py
  route.py           # 10 endpoints
  queries.py          # All DB queries (CRUD, usage, stats)
  evaluation.py       # Rule engine (12 operators, recursive all/any)
  request_model.py    # Pydantic request models
  response_model.py   # Pydantic response models
```

**Auth**: All endpoints require `require_pbi_or_retention()`.
**Collections**: `sales_coupons`, `sales_coupon_usage` (prefixed to avoid clash with CRM `coupons` collection).

---

## 1. Coupon CRUD

### Create
```
Admin opens /pricing/coupons → "Create Coupon" button
  → Fill: title, code (optional, auto-gen CPN-{n}), discount type/value, limits, validity
  → Optionally add advanced conditions via visual builder
  → POST /coupons/create
  → Coupon appears in management tab
```

### List & Filter
```
GET /coupons/list?page=1&limit=20&search=SUMMER&is_active=true
  → Paginated, sorted by created_at DESC
  → Search matches coupon_code or title (case-insensitive regex)
```

### Update (Versioned)
```
PUT /coupons/{coupon_id}
  → Increments version number
  → Appends to change_log: { changed_by, changed_at, changes: {field: {old, new}} }
  → Only changed fields are logged
```

### Delete (Soft)
```
DELETE /coupons/{coupon_id}
  → Sets is_active = false (doesn't remove document)
```

---

## 2. Eligibility Check

### Evaluation Flow (8 steps)
```
POST /coupons/check-eligibility
  {coupon_code, order_total, order_quantity, product_skus, category_names,
   customer_id, customer_state, customer_category, payment_mode}

  1. Find coupon by code → check is_active
  2. Check valid_from / valid_to dates
  3. Check overall_limit vs usage_count
  4. Check per_customer_limit vs customer's historical usage
  5. Check per_agent_limit vs agent's monthly usage
  6. Check scope_type: product SKUs or category names overlap with cart
  7. Check flat filters: min_order_value, min_order_quantity, states, payment, customer_category
  8. Evaluate nested conditions via rule engine (if present)
  9. Calculate discount:
     - fixed: min(discount_value, order_total)
     - percentage: order_total × discount_value / 100, capped at discount_upto

  → Return: {eligible: true/false, discount_amount, reason}
```

### Rule Engine (evaluation.py)
- **No external dependencies** — 60 lines of Python
- Supports `all` (AND) and `any` (OR) grouping, infinitely nestable
- 12 operators: equal_to, not_equal_to, greater_than, greater_than_or_equal_to, less_than, less_than_or_equal_to, contains, not_contains, in, not_in, shares_at_least_one, contains_all
- Context dict built from order/customer data, supports dot notation for nested values
- Type-safe: catches TypeError/ValueError and returns False

### Example Condition (stored in MongoDB)
```json
{
  "all": [
    {"name": "product_skus_in_cart", "operator": "shares_at_least_one", "value": ["SKU-NEEM-OIL-1L"]},
    {"name": "order_total", "operator": "greater_than_or_equal_to", "value": 2000},
    {"any": [
      {"name": "customer_state", "operator": "equal_to", "value": "Maharashtra"},
      {"name": "customer_state", "operator": "equal_to", "value": "Karnataka"}
    ]}
  ]
}
```
Reads as: Cart has Neem Oil **AND** order >= ₹2000 **AND** customer is in MH **OR** KA.

---

## 3. Apply Coupon

```
POST /coupons/apply
  {coupon_code, order_id, order_total, order_quantity, product_skus,
   category_names, customer_id, customer_state, customer_category, payment_mode}

  → Runs full eligibility check (same 8 steps)
  → If eligible:
    → Insert record into sales_coupon_usage
    → Increment sales_coupons.usage_count
    → Return {discount_applied, usage_id}
  → If not eligible:
    → Return 400 with reason
```

---

## 4. Usage Logs

```
GET /coupons/usage-logs?page=1&limit=20&coupon_code=CPN-1&start_date=2026-03-01&end_date=2026-03-31
  → Paginated redemption history
  → Joins coupon title from sales_coupons collection
  → Sorted by applied_at DESC
```

---

## 5. Stats

```
GET /coupons/stats
  → total_coupons: count of sales_coupons
  → active_coupons: count where is_active=true
  → total_usage: count of sales_coupon_usage
  → total_discount_given: sum of discount_applied
```

---

## 6. Rule Metadata

```
GET /coupons/rule-metadata
  → Returns available operators (12) and variables (9) for the frontend condition builder UI
  → Each operator has: value, label, supported types
  → Each variable has: name, label, type (number/string/list)
```

---

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/coupons/create` | Create coupon (auto-gen code if empty) |
| GET | `/coupons/list` | Paginated list (search, is_active filter) |
| GET | `/coupons/stats` | Dashboard stat cards |
| GET | `/coupons/usage-logs` | Redemption logs (date/code filters) |
| GET | `/coupons/rule-metadata` | Operators + variables for UI builder |
| GET | `/coupons/{coupon_id}` | Single coupon details |
| PUT | `/coupons/{coupon_id}` | Update with versioned changelog |
| DELETE | `/coupons/{coupon_id}` | Soft delete (is_active=false) |
| POST | `/coupons/check-eligibility` | Test coupon against order context |
| POST | `/coupons/apply` | Apply coupon + record usage |

---

## Frontend

### Page: Coupons.tsx — `/pricing/coupons`
- **Hook**: `useCoupons()` → 7 methods (fetchCoupons, fetchStats, fetchUsageLogs, fetchRuleMetadata, createCoupon, updateCoupon, deleteCoupon)
- **Stat cards** (4): Total Coupons, Active, Total Usage, Total Discount Given
- **Tabs**: Coupon Management | Redemption Logs
- **Management tab columns**: Coupon Code (+ title), Discount (type + value + cap), Min Order, Usage (count/limit + progress bar), Validity, Status, 3-dot menu
- **3-dot menu**: View Details, Activate/Deactivate, Delete
- **View Details dialog**: Full coupon info in sections:
  - Header: code, title, status badge
  - Discount info: type, value, max cap, min order, scope
  - Limits: overall, per-customer, per-agent (shows "Unlimited" for -1)
  - Usage: count/limit with progress bar
  - Validity: from/to dates
  - Filters: allowed/blocked states (badges), payment modes, customer categories, applicable products/categories
  - Advanced Conditions: rendered rule tree with color-coded borders (blue=AND, orange=OR), badges for variable names
  - Metadata: created by (full-width row for long emails), version, created/updated timestamps
- **Redemption tab columns**: Coupon Code (+ title), Order ID, Agent, Customer, Discount, Order Value, Applied At

### Component: CreateCouponDialog.tsx
- **Fields**: Title*, Code (auto if empty), Description, Discount Type/Value*, Max Discount, Min Order, Overall Limit, Per Customer Limit, Valid From/To, Scope
- **Visual Condition Builder** (toggle):
  - Add condition groups (each group has AND/OR selector)
  - Each condition row: Variable dropdown (from `/rule-metadata`), Operator dropdown, Value input (supports comma-separated for lists)
  - Add/remove conditions per group, add/remove groups
  - Builds nested `all`/`any` JSON on submit
- **Validation**: title + discount_value required

### Hook: useCoupons.ts
```typescript
const {
  coupons, usageLogs, stats, ruleMetadata, loading, error,
  totalCoupons, totalLogs,
  fetchCoupons, fetchStats, fetchUsageLogs, fetchRuleMetadata,
  createCoupon, updateCoupon, deleteCoupon,
} = useCoupons();
```

---

## Discount Calculation

| Type | Formula | Cap |
|------|---------|-----|
| `fixed` | `min(discount_value, order_total)` | Cannot exceed order total |
| `percentage` | `order_total × discount_value / 100` | `min(calculated, discount_upto)` if cap set |

---

## Comparison with Freebie System (Inventory-Management-Backend)

| Aspect | Freebies | Coupons |
|--------|----------|---------|
| Stack | Node.js, json-rules-engine | Python, custom evaluator |
| Output | Free product added to order | Discount on existing items |
| Trigger | Automatic on order | Manual (agent enters code) |
| Quotas | Agent + customer monthly (separate collection) | Per-coupon limits (in coupon doc) |
| Conditions | json-rules-engine facts/rules | Custom all/any nesting |
| Scope | Product/category via rules | Explicit scope_type field + nested conditions |
| Versioning | Rule versioning + usage logs | Version number + change_log array |

#active #auth-required
