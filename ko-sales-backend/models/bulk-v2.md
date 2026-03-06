---
model: BulkV2
collection: bulk_v2
status: active
last_updated: 2026-03-06
tags: [catalog, associate, products, crud]
depends_on: []
referenced_by: [associate-rule, associate-service, visibility-rule-engine, pricing-rule-engine, bulks-module]
created: 2026-02-25
---

# BulkV2

Product catalog for B2B associates. Each bulk represents a parent product with multiple product variants (sizes).

## Fields

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| sku | String | Yes | Unique, uppercase. **NOT `bulk_sku`** |
| trade_name | String | Yes | Display name |
| technical_name | String | No | Chemical/scientific name |
| category | String | No | **NOT ObjectId** — plain string, no `.populate()` |
| sub_category | String | No | Plain string |
| channels | Object | No | `{ website, distributor, retailer, associate }` — all booleans, default false |
| description | Object | No | `{ english, hindi }` — nested, not separate fields |
| gst_rate | String | No | **String in DB**, not Number |
| hsn_code | String | No | HSN classification code |
| bulk_min_order_qty | String | No | **String in DB** |
| dist_min_order_qty | String | No | **String in DB** |
| technicals | Array | No | `[{ technical_sku, technical_name, value, unit }]` |
| crops | String[] | No | Target crops |
| diseases | String[] | No | Target diseases |
| dosage | String[] | No | Dosage instructions |
| features | String[] | No | Product features |
| applications | String[] | No | Application methods |
| bulk_type | String | No | e.g. EC, WP, SL |
| uom | String | No | e.g. l, kg, ml |
| sku_type | String | No | SKU classification |
| is_active | Boolean | No | Default true. Soft delete flag |
| created_at | Date | Auto | Mongoose timestamps |
| updated_at | Date | Auto | Mongoose timestamps |

## Key Rules

- Schema has `strict: false` — allows extra DB fields not in schema
- Filter for associate visibility: `channels.associate: { $ne: false }` (handles missing field in legacy docs)
- 1134 documents in production (migrated from CRM-Database)
- SKU always uppercased on lookup (`sku.toUpperCase()`)
- Duplicate SKU check on create (returns 409 Conflict)

## CRUD Module (`src/modules/bulks/`)

Added 2026-03-06. Direct catalog management separate from associate visibility engine.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/bulks` | Create bulk (SKU duplicate check → 409) |
| GET | `/api/v1/bulks` | List — paginated, search (trade_name/technical_name/sku), filters |
| GET | `/api/v1/bulks/categories` | Distinct category names (sorted) |
| GET | `/api/v1/bulks/categories/:category/sub-categories` | Sub-categories for a category (sorted) |
| GET | `/api/v1/bulks/:sku` | Get single bulk by SKU |
| PUT | `/api/v1/bulks/:sku` | Update bulk by SKU |
| DELETE | `/api/v1/bulks/:sku` | Hard delete by SKU |

### Query Parameters (GET /bulks)

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| page | Number | 1 | Page number |
| limit | Number | 20 | Items per page |
| search | String | — | Regex on trade_name, technical_name, sku |
| category | String | — | Exact match |
| sub_category | String | — | Exact match |
| bulk_type | String | — | Exact match |
| uom | String | — | Exact match |
| channel | String | — | Filter by `channels.<channel>: true` |
| is_active | Boolean | — | Filter active/inactive |

### Response Format (List)
```json
{
  "success": true,
  "data": [...],
  "total": 1134,
  "page": 1,
  "total_pages": 57
}
```

### Files

| File | Purpose |
|------|---------|
| `bulks.module.ts` | Module (exports BulksService) |
| `bulks.controller.ts` | 7 endpoints, Swagger decorated |
| `bulks.service.ts` | CRUD + categories + sub-categories |
| `dto/create-bulk.dto.ts` | Nested DTOs: ChannelsDto, DescriptionDto, TechnicalDto |
| `dto/update-bulk.dto.ts` | PartialType(CreateBulkDto) + is_active |
| `dto/get-bulks.dto.ts` | Pagination + filters |
| `schemas/bulk-v2.schema.ts` | Mongoose schema (strict: false, timestamps) |

## Related Collections

| Collection | Relationship |
|-----------|-------------|
| products_v2 | Products have `bulk.sku` referencing this collection |
| bulk_default_pricing | Pricing fallback by `sku` |
| associate_partner_rules | Rules target bulks via `sku` or `categories` |

## Two Access Patterns

1. **Direct CRUD** (`/api/v1/bulks/*`) — BulksModule: admin catalog management, no visibility rules
2. **Associate Catalog** (`/api/v1/associate/bulks/*`) — AssociateModule: visibility-filtered, pricing, packaging, images
