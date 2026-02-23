# Pincode Case Pricing — MongoDB Models

`#active`

---

## pincode_case_pricing Collection

Zone-based pricing overrides for product cases. A case can have different prices in different zones, identified by pincodes.

### Schema

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| _id | ObjectID | PK | Auto-generated |
| case_sku | string | NOT NULL | Links to `cases.case_id` |
| zone | string | NOT NULL | Geographic zone (e.g., "North", "Metro", "South") |
| uom | string | NOT NULL | Unit of measurement: l, ml, gm, kg (normalized) |
| pincodes_array | []string | NOT NULL | Array of pincode strings (e.g., ["110001", "110002"]) |
| pricing | []PricingDetail | NOT NULL | Quantity-based pricing tiers |
| is_active | bool | default: false | Enable/disable without deletion |
| created_at | Date | Auto | |
| updated_at | Date | Auto | |
| created_by | string | | User email from JWT |
| updated_by | string | | User email from JWT |

### PricingDetail (embedded)

| Field | Type | Description |
|-------|------|-------------|
| quantity | int | Units in this tier |
| unit_price | float64 | Price per unit |
| uom_unit_price | float64 | Price per UOM (unit_case_price / case_qty / variant_value) |
| unit_case_price | float64 | Total price for one case at this quantity |
| default | bool | Auto-set: true on 2nd tier (or 1st if only one) |

### Indexes

| Index | Fields | Type | Purpose |
|-------|--------|------|---------|
| Unique | `(case_sku, pincodes_array)` | Unique compound | Prevents same pincode in multiple zones per SKU |
| Compound | `(case_sku, zone)` | Regular | Fast zone lookups |
| Compound | `(is_active, created_at)` | Regular | Active filtering with sort |

### Price Validation Rules

Prices must be mathematically consistent (±0.01 tolerance):
```
unit_price = unit_case_price / case_qty
uom_unit_price = unit_case_price / (case_qty * variant_value)
```

Where:
- `case_qty` = `cases.quantity` (units per case)
- `variant_value` = parsed from `cases.variant` (e.g., "500 ml" → 0.5 if base UOM is "l")

---

## pincode_case_audit_log Collection

Immutable audit trail for all pincode pricing changes.

### Schema

| Field | Type | Description |
|-------|------|-------------|
| _id | ObjectID | PK |
| pricing_id | ObjectID | Reference to pincode_case_pricing._id |
| case_sku | string | Denormalized for filtering |
| zone | string | Denormalized for filtering |
| action | string | "CREATE" or "UPDATE" |
| changed_by_email | string | User email from JWT |
| changed_at | Date | Timestamp |
| fields_changed | []string | List of modified field names |
| new_values | map[string]any | New values for changed fields only |
| platform | string | From JWT platform claim (web, mobile) |

### Index

| Fields | Type | Purpose |
|--------|------|---------|
| `(pricing_id, changed_at -1)` | Compound | Fast retrieval sorted newest first |

---

## Related: cases Collection (external)

The pincode case system depends on these fields from the `cases` collection:

| Field | Used For |
|-------|----------|
| `case_id` | Matched against `case_sku` |
| `quantity` | case_qty for price validation |
| `uom` | Base UOM for variant_value calculation |
| `variant` | Parsed for variant_value (e.g., "500 ml") |
| `is_available` | Must be true to create pricing |
| `has_pincode_case` | Set to true when pricing is created |

---

## Entity Relationship

```
cases (Inventory-Mgmt-Backend writes, utilities-go reads)
  │
  └── case_id ←── case_sku ── pincode_case_pricing (utilities-go writes)
                                    │
                                    └── _id ←── pricing_id ── pincode_case_audit_log
```

**Cross-service pattern:**
- **utilities-go** manages pincode_case_pricing (CRUD via REST API)
- **Inventory-Management-Backend** reads pincode_case_pricing directly from MongoDB (search product API)
- Both share the same MongoDB database (`CRM-Database`)
