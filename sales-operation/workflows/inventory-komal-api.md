# Inventory Module — Komal API Integration

## Overview
The Inventory module proxies the Komal warehouse management system (`api.komal.ko-tech.in`) instead of querying MongoDB directly. All inventory data comes from Komal in real-time.

## Architecture
```
Frontend (OutOfStock.tsx)
  → useInventory hook       → GET /api/inventory/inventory
  → useInventoryStats hook  → GET /api/inventory/stats
      ↓
Backend (FastAPI)
  → Inventory/route.py   (3 endpoints)
  → Inventory/queries.py (Komal API client)
      ↓
Komal API (api.komal.ko-tech.in)
  → OAuth2 login → Bearer token (cached 9h)
  → GET /inventory/{store_name} (paginated, filterable)
```

## Authentication
- **OAuth2 password grant**: `POST /login` with form data (`username`, `password`, `grant_type=password`, `scope=""`)
- Returns `access_token` (valid ~10h, cached for 9h in `_token_cache` dict)
- Credentials in `.env`: `KOMAL_BASE_URL`, `KOMAL_USERNAME`, `KOMAL_PASSWORD`, `KOMAL_DEFAULT_STORE`
- Token auto-refreshes on expiry

### Environment Variables
```
KOMAL_BASE_URL=https://api.komal.ko-tech.in
KOMAL_USERNAME=su@admin.sql
KOMAL_PASSWORD=ghbvfty123
KOMAL_DEFAULT_STORE=Gilehri Warehouse
```

## Komal API — Inventory Endpoint
```
GET /inventory/{store_name}
Query params:
  - page (int)      — 1-indexed
  - page_size (int)  — items per page (max 500 for batch, default 25)
  - status (string)  — "negative" | "low" | "high"
  - filters (JSON string) — array of field filters
```

### Native Filters (key discovery)
Komal API accepts a `filters` query param as a JSON string:
```json
[
  {
    "field": "category",
    "filters": [{"operator": "contains", "operator_value": null, "value": "FG"}]
  },
  {
    "field": "name",
    "filters": [{"operator": "contains", "operator_value": null, "value": "neem"}]
  }
]
```
- **Supported fields**: `category`, `name`, `sku` (tested and confirmed)
- **Operator**: `contains` (case-insensitive substring match)
- **Multiple filters** are AND-ed together
- This eliminates the need for server-side caching/filtering — Komal handles pagination + filtering natively

### Status Mapping
| Komal Status | Our Status | Meaning |
|---|---|---|
| `negative` | `out_of_stock` | Stock ≤ 0 |
| `low` | `low_stock` | Below min_stock |
| `high` | `available` | Adequate stock |

## Backend Endpoints

### GET /inventory/inventory
Main inventory listing. Proxies to Komal with optional filters.

**Query params**: `page`, `limit`, `status` (out_of_stock/low_stock/available), `store`, `category`, `search`

**Backend logic** (`queries.py` → `fetch_komal_inventory`):
1. Maps our status names to Komal status (`reverse_status_map`)
2. Builds `filters` JSON from category and search params via `_build_filters()`
3. Single GET to Komal API with all params
4. Transforms each item via `_transform_item()` (maps fields, adds computed status)
5. Returns `InventoryPaginatedResponse`

### GET /inventory/stats
Returns counts for stat cards. Makes 3 separate calls to Komal (one per status) with `page_size=1` to get just the `total` from each.

**Response**: `{ out_of_stock_count, low_stock_count, available_count, total }`

Respects category and search filters (passes same `filters` JSON to all 3 calls).

### GET /inventory/stores
Returns hardcoded list of 7 warehouse names (from Komal JWT claims):
- **Gilehri Warehouse** (~3931 items, default)
- **Katyayani Organics** (~1629 items)
- Gilehri B, Gilehri International, Gilehri W3, Gilehri W4, Gilehri Workstation 2

## Data Model (InventoryItem)

**Fields from Komal**: `id`, `sku`, `name`, `prod_name`, `tech_name`, `category`, `uom`, `min_stock`, `labeled_qty`, `unlabeled_qty`, `loose_qty`, `room_no`, `rack_no`, `shelf_no`, `store`, `status` (komal raw), `hold_labeled_qty`, `hold_unlabeled_qty`

**Added by backend**: `status` (mapped to our naming), `komal_status` (raw value preserved)

**Categories in Komal**: FG (finished goods), PKG (packaging), LABEL, STICKER, RM (raw material), and others

## Frontend

### OutOfStock.tsx
- **Store selector** dropdown (7 warehouses)
- **Status filter** (All / Out of Stock / Low Stock / Available) — server-side via Komal `status` param
- **Category filter** (All / FG / PKG / LABEL / STICKER / RM / Other) — server-side via Komal `filters` param
- **Search input** with 500ms debounce — server-side via Komal `filters` param (searches `name` field)
- **Stats cards**: Total Items, Out of Stock, Low Stock, Available — from `/inventory/stats`
- **Table columns**: Product (name+SKU), Category, Stock Qty (color-coded), Min Stock, UOM, Location, Status badge
- **Server-side pagination** with page/limit controls

### Hooks
- `useInventory(params)` — fetches paginated inventory from `/inventory/inventory`
- `useInventoryStats(store, category?, search?)` — fetches counts from `/inventory/stats`

## Performance
- Each request is a **single API call** to Komal (~1s response time)
- Stats endpoint makes **3 calls** (~2s total)
- **No caching needed** — Komal handles filtering and pagination natively
- Previous approach (fetching all ~4000 items + server-side filtering + 5min cache) was removed after discovering the `filters` param

## Files
| File | Purpose |
|---|---|
| `Inventory/queries.py` | Komal API client (auth, filters, fetch, counts) |
| `Inventory/route.py` | 3 FastAPI endpoints (inventory, stats, stores) |
| `Inventory/response_model.py` | Pydantic models (InventoryItem, InventoryPaginatedResponse) |
| `src/hooks/useInventory.ts` | useInventory + useInventoryStats hooks |
| `src/pages/inventory/OutOfStock.tsx` | Full inventory page UI |
