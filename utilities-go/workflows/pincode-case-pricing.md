# Pincode Case Pricing — API & Workflow

`#active` `#auth-required` `#cross-project`

---

## Purpose

Enable zone-based pricing for product cases. Different geographical zones (Metro, North, South, etc.) can have different pricing tiers for the same case. Pincodes map to zones; the search product API uses retailer's pincode to serve zone-specific prices.

---

## API Reference

**Base path:** `/pincode-cases` | **Auth:** JWKS JWT required

### Create Pricing
`POST /pincode-cases`

```json
{
  "case_sku": "CS-10-ABC123",
  "zone": "North",
  "uom": "l",
  "pincodes_array": ["110001", "110002", "110003"],
  "pricing": [
    {
      "quantity": 10,
      "unit_price": 100.00,
      "uom_unit_price": 95.00,
      "unit_case_price": 950.00
    },
    {
      "quantity": 20,
      "unit_price": 90.00,
      "uom_unit_price": 85.50,
      "unit_case_price": 855.00
    }
  ],
  "is_active": true
}
```

**Validation:**
- Case must exist in `cases` collection with `is_available: true` and `quantity > 0`
- UOM must be l, ml, gm, or kg (auto-normalized from ltr/liter/gram/kilo etc.)
- All pincodes must be numeric strings
- Pricing math must be consistent (unit_price = unit_case_price / case_qty, ±0.01)
- 409 Conflict if pincode already mapped to another zone for same case_sku

**Side effects:**
- Sets `has_pincode_case: true` on the case document
- Creates async audit log entry (action: "CREATE")

### List Pricing
`GET /pincode-cases?page=1&page_size=10&case_sku=CS-10-ABC123&zone=North&pincode=110001&is_active=true`

All filters optional. Paginated (max 100/page).

### Get Single Entry
`GET /pincode-cases/:id`

### Update Pricing
`PUT /pincode-cases?id=ObjectID`
or
`PUT /pincode-cases?case_sku=CS-10-ABC123&zone=North`

All fields optional. Only changed fields are updated. Re-validates pricing math if pricing changed. Creates audit log with changed fields only.

### Check Pincode Availability
`GET /pincode-cases/check-pincode?sku=CS-10-ABC123&pincode=110001`

Returns whether pincode is already mapped for that SKU and which zone.

### Cases Summary (Aggregation)
`GET /pincode-cases/cases-summary?page=1&page_size=10&sku=PROD001&name=Sunflower&pincode=110001&zone=North`

Returns cases with zone count statistics. Cannot mix product filters (sku, name) with distribution filters (pincode, zone).

```json
{
  "summary": {"total_products": 125, "total_zones": 8, "total_pincodes": 2450},
  "data": [
    {"sku": "PROD001", "case_id": "CS-10-ABC123", "product_name": "Sunflower Oil", "zone_count": 5}
  ]
}
```

### Zone Pricing Details
`GET /pincode-cases/zone-pricing?case_sku=CS-10-ABC123&zone=North`

Full pricing breakdown including calculated `case_price = case_qty * quantity * unit_price` and fallback prices from main case.

### Distinct Zones
`GET /pincode-cases/distinct/zones?case_sku=CS-10-ABC123&is_active=true`

Returns list of unique zone names.

### Audit Logs
`GET /pincode-cases/:id/audit-logs`

Returns all change history for a pricing entry, newest first.

---

## Consumer: Search Product API (Inventory-Management-Backend)

**Route:** `GET /cases/search/product`
**File:** `Inventory-Management-Backend/routes/cases/controllers/cases-controller.js`

### How It Works

```
Client → GET /cases/search/product?pincode=110001&category=Oils&...
    │
    ├── Resolve pincode (from query param or Retailer.address.pincode)
    │
    ├── Find matching products (category, subcategory, displayName, sku filters)
    │
    └── FOR EACH CASE on each product:
        │
        ├── IF retailer_pincode provided:
        │   └── pincode_case_pricing.findOne({
        │         is_active: true,
        │         pincodes_array: { $in: [retailer_pincode] },
        │         case_sku: case_id
        │       })
        │
        ├── IF pincode pricing found:
        │   ├── Use pincode pricing tiers for unit_price, uom_unit_price, unit_case_price
        │   ├── case_price = pincode pricing's unit_case_price
        │   └── Format: per_unit_price = "{unit_price}/{variant}"
        │
        └── ELSE (no pincode pricing):
            ├── Use default case pricing from cases collection
            └── Format: per_unit_price = UOM-based calculation
```

### Integration Type

**Direct MongoDB query** — NOT an HTTP call to utilities-go. Both services share the same `CRM-Database` MongoDB instance. Inventory-Management-Backend uses a generic Mongoose schema (`strict: false`) to read `pincode_case_pricing`.

### Pincode Resolution

1. `?pincode=XXXXX` query parameter (preferred)
2. Fallback: `Retailer.findOne({ contact: '+91{retailer}' })` → `address.pincode`

### Pricing Priority

| Priority | Source | When Used |
|----------|--------|-----------|
| 1 | pincode_case_pricing | Pincode matches an active entry for the case |
| 2 | cases.pricing (default) | No pincode provided or no matching pincode pricing |

### Margin Calculation

```javascript
margin = (1 - (unit_price / md_price)) * 100
```

Applied to each pricing tier in the response.

---

## UOM Conversion Table

| Input | Normalized To |
|-------|--------------|
| ltr, liter, litre, L | l |
| gram, g | gm |
| kilo, kilogram | kg |
| ml | ml |

**Variant value calculation examples:**
- Base UOM "l", variant "500 ml" → variant_value = 0.5
- Base UOM "kg", variant "250 gm" → variant_value = 0.25
- Base UOM "l", variant "1 L" → variant_value = 1.0

---

## Business Rules

1. **Unique pincode per zone per SKU** — Enforced by unique compound index
2. **No hard delete** — Use `is_active: false` for deactivation
3. **Immutable audit trail** — Every create/update logged, no edit/delete of logs
4. **Case must be available** — `is_available: true` required for pricing creation
5. **Math consistency** — All three price levels must be derivable from each other
6. **Default tier** — 2nd pricing tier auto-marked as default (or 1st if only one)

---

## Files

### utilities-go (CRUD API)
| File | Purpose | Lines |
|------|---------|-------|
| handlers/pincode_cases.go | 9 handlers + 9 helpers | ~1330 |
| models/pincode_case_model.go | Models, request/response structs | ~90 |
| routes/pincodeCaseRoutes.go | Route registration | ~40 |
| scripts/test-pincode-api.sh | API test script (12 test cases) | — |
| docs/pincode_case_pricing_docs/ | API docs + quickstart | — |

### Inventory-Management-Backend (Consumer)
| File | Purpose |
|------|---------|
| routes/cases/route.js | Route: `GET /search/product` |
| routes/cases/controllers/cases-controller.js | searchProduct handler (lines 109-570) |
| routes/cases/utils/casesMarginCalculation.js | Margin + UOM price formatting |
