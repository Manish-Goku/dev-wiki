---
workflow: Bulk Catalog CRUD
status: active
last_worked: 2026-03-06
tags: [#active, #crud, #catalog]
models: [bulk-v2]
modules: [bulks]
created: 2026-03-06
---

# Bulk Catalog CRUD — In-Depth Technical Reference

Direct CRUD module for the `bulk_v2` collection. Provides admin-level product catalog management, separate from the associate visibility engine (`/associate/bulks/*`).

## Module Location

`src/modules/bulks/` — 7 files: schema, 3 DTOs, service, controller, module.

---

## 1. Architecture

### Two Access Patterns for BulkV2

The `bulk_v2` collection is accessed by two independent modules:

| Module | Route Prefix | Purpose | Auth |
|--------|-------------|---------|------|
| **BulksModule** | `/api/v1/bulks` | Direct CRUD — create, read, update, delete | JWT (global guard) |
| **AssociateModule** | `/api/v1/associate/bulks` | Visibility-filtered catalog with pricing, packaging, images | `@Public()` |

They share the same MongoDB collection but have different schemas registered:
- `BulksModule` uses `BulkV2Schema` with `strict: false` and `is_active` field
- `AssociateModule` uses its own schema registration via `MongooseModule.forFeature()`

### Module Wiring

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([{ name: BulkV2.name, schema: BulkV2Schema }]),
  ],
  controllers: [BulksController],
  providers: [BulksService],
  exports: [BulksService],  // available for other modules to import
})
export class BulksModule {}
```

Registered in `app.module.ts` imports array. Swagger tag: `Bulks`.

---

## 2. Schema (`schemas/bulk-v2.schema.ts`)

```typescript
@Schema({ collection: 'bulk_v2', timestamps: true, strict: false })
export class BulkV2 {
  @Prop({ required: true, unique: true, uppercase: true })
  sku: string;

  @Prop({ required: true })
  trade_name: string;

  @Prop() technical_name: string;
  @Prop() category: string;         // plain String, NOT ObjectId
  @Prop() sub_category: string;     // plain String
  @Prop({ type: Object }) channels: Record<string, boolean>;
  // { website, distributor, retailer, associate } — all booleans
  @Prop({ type: Object }) description: Record<string, string>;
  // { english, hindi }
  @Prop() gst_rate: string;         // String in DB, NOT Number
  @Prop() hsn_code: string;
  @Prop() bulk_min_order_qty: string;  // String in DB
  @Prop() dist_min_order_qty: string;  // String in DB
  @Prop({ type: [Object] }) technicals: Record<string, any>[];
  // [{ technical_sku, technical_name, value, unit }]
  @Prop({ type: [String] }) crops: string[];
  @Prop({ type: [String] }) diseases: string[];
  @Prop({ type: [String] }) dosage: string[];
  @Prop({ type: [String] }) features: string[];
  @Prop({ type: [String] }) applications: string[];
  @Prop() bulk_type: string;
  @Prop() uom: string;
  @Prop() sku_type: string;
  @Prop({ default: true }) is_active: boolean;
}
```

Key points:
- `strict: false` — extra fields from legacy DB docs pass through
- `timestamps: true` → `created_at`, `updated_at` (via `timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' }`)
- `sku` has `uppercase: true` transform
- 1134 documents in production

---

## 3. DTOs

### CreateBulkDto (`dto/create-bulk.dto.ts`)

Nested DTO structure for validated creation:

```
CreateBulkDto
├── sku (required, string)
├── trade_name (required, string)
├── technical_name (optional)
├── category (optional)
├── sub_category (optional)
├── channels: ChannelsDto (optional)
│   ├── website (boolean, default false)
│   ├── distributor (boolean, default false)
│   ├── retailer (boolean, default false)
│   └── associate (boolean, default false)
├── description: DescriptionDto (optional)
│   ├── english (string)
│   └── hindi (string)
├── gst_rate (string)
├── hsn_code (string)
├── bulk_min_order_qty (string)
├── dist_min_order_qty (string)
├── technicals: TechnicalDto[] (optional)
│   ├── technical_sku (string)
│   ├── technical_name (string)
│   ├── value (string)
│   └── unit (string)
├── crops (string[])
├── diseases (string[])
├── dosage (string[])
├── features (string[])
├── applications (string[])
├── bulk_type (string)
├── uom (string)
└── sku_type (string)
```

All validation via `class-validator` + `class-transformer`. Nested objects use `@ValidateNested()` + `@Type(() => NestedDto)`.

### UpdateBulkDto (`dto/update-bulk.dto.ts`)

`PartialType(CreateBulkDto)` + explicit `is_active: boolean` field. All fields optional.

### GetBulksDto (`dto/get-bulks.dto.ts`)

| Field | Type | Default | Validation |
|-------|------|---------|------------|
| page | number | 1 | `@Min(1)`, `@Type(() => Number)` |
| limit | number | 20 | `@Min(1)`, `@Type(() => Number)` |
| search | string | — | Regex on trade_name, technical_name, sku |
| category | string | — | Exact match |
| sub_category | string | — | Exact match |
| bulk_type | string | — | Exact match |
| uom | string | — | Exact match |
| channel | string | — | Maps to `channels.<channel>: true` filter |
| is_active | boolean | — | `@Transform` handles string `"true"`/`"false"` |

---

## 4. Service Methods (`bulks.service.ts`)

### create(dto)
1. Check if SKU exists (case-insensitive via `toUpperCase()`)
2. If exists → throw `ConflictException` (409)
3. Create document via `bulk_model.create(dto)`

### find_all(dto)
1. Build filter from query params:
   - `search` → `$or: [{ trade_name: regex }, { technical_name: regex }, { sku: regex }]`
   - `category`, `sub_category`, `bulk_type`, `uom` → exact match
   - `channel` → `channels.<channel>: true`
   - `is_active` → exact boolean match
2. Parallel: `find()` with sort/skip/limit + `countDocuments()`
3. Return: `{ success, data, total, page, total_pages }`

Sort: `created_at: -1` (newest first)

### find_by_sku(sku)
- `findOne({ sku: sku.toUpperCase() }).lean()`
- Throws `NotFoundException` if not found

### update(sku, dto)
- `findOneAndUpdate({ sku: sku.toUpperCase() }, { $set: dto }, { new: true, lean: true })`
- Throws `NotFoundException` if not found

### delete(sku)
- `deleteOne({ sku: sku.toUpperCase() })`
- Checks `deletedCount === 0` → throws `NotFoundException`
- **Hard delete** (not soft delete)

### get_categories()
- `distinct('category', { category: { $ne: null } })`
- Returns sorted array

### get_sub_categories(category)
- `distinct('sub_category', { category, sub_category: { $ne: null } })`
- Returns sorted array

---

## 5. Controller Endpoints (`bulks.controller.ts`)

All endpoints under `@Controller('bulks')` with `@ApiTags('Bulks')` and `@ApiBearerAuth()`.

| Method | Path | Handler | Response |
|--------|------|---------|----------|
| POST | `/bulks` | `create(@Body() dto)` | `{ success, data }` |
| GET | `/bulks` | `find_all(@Query() dto)` | `{ success, data[], total, page, total_pages }` |
| GET | `/bulks/categories` | `get_categories()` | `{ success, data[] }` |
| GET | `/bulks/categories/:category/sub-categories` | `get_sub_categories(@Param)` | `{ success, data[] }` |
| GET | `/bulks/:sku` | `find_by_sku(@Param)` | `{ success, data }` |
| PUT | `/bulks/:sku` | `update(@Param, @Body() dto)` | `{ success, data }` |
| DELETE | `/bulks/:sku` | `delete(@Param)` | `{ success }` |

**Route ordering matters:** `/bulks/categories` is defined before `/bulks/:sku` to avoid NestJS treating "categories" as a SKU parameter.

---

## 6. Error Handling

| Scenario | Exception | Code |
|----------|-----------|------|
| Duplicate SKU on create | `ConflictException` | 409 |
| SKU not found (get/update/delete) | `NotFoundException` | 404 |
| Invalid DTO fields | `BadRequestException` (ValidationPipe) | 400 |

---

## 7. Test Results (2026-03-06)

All endpoints tested against production data (1134 documents).

| Test | Result |
|------|--------|
| `GET /bulks` | 1134 total, 57 pages, returns page 1 with 20 items |
| `GET /bulks/BK-1181` | Returns correct bulk with trade_name, category |
| `GET /bulks/categories` | Returns 5+ categories: bulk, fertilizer, fungicide, herbicide, manure |
| `GET /bulks/categories/fungicide/sub-categories` | Returns 3: bio fungicide, chemical fungicide, organic fungicide |
| `GET /bulks?search=neem` | 13 matching results |
| `GET /bulks?category=fungicide` | 71 matching results |
| `GET /bulks?category=fungicide&limit=2` | Correct pagination with 2 items |

---

## 8. Data Quality Notes (Production)

Based on 1134 documents in `bulk_v2`:

| Field | Fill Rate | Notes |
|-------|-----------|-------|
| sku | 100% | All unique, uppercase |
| trade_name | ~99% | Most have names |
| category | ~95% | 5+ distinct values |
| sub_category | ~80% | Varies by category |
| bulk_type | ~5% | Mostly empty |
| description | ~5% | Very few have both english/hindi |
| features | ~1% | Almost all empty arrays |
| applications | ~1% | Almost all empty arrays |
| gst_rate | ~70% | String values like "18", "5" |
| channels | ~60% | Many legacy docs missing this entirely |

---

## 9. Branch History

- **2026-03-06:** Created on `manish-master-v2` (commit `daedd73`), merged into `praveen`
- Both branches now have the bulks CRUD module
