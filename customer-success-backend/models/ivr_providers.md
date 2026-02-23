# ivr_providers

Dynamic IVR provider registry. Each row = one IVR department. Replaces all hardcoded department constants. Admin-managed via Masters page.

## Schema

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| `id` | uuid (PK) | `gen_random_uuid()` | |
| `slug` | text (UNIQUE) | — | Join key to `ivr_calls.department`. URL-safe identifier. |
| `name` | text | — | Display label (e.g. "General IVR", "Manish IVR") |
| `api_key` | text (UNIQUE) | `gen_random_uuid()` | Webhook authentication key |
| `field_mapping` | jsonb | `{}` | Maps webhook JSON paths → `ivr_calls` columns (dot-notation) |
| `status_map` | jsonb | `{}` | Maps provider status strings → canonical statuses |
| `icon` | text | `'Phone'` | Lucide icon name for sidebar/UI |
| `did_numbers` | text[] | `'{}'` | DID numbers assigned to this provider |
| `display_order` | int | `0` | Sidebar + tab ordering (ascending) |
| `is_active` | boolean | `true` | Soft-delete flag. Inactive = hidden from sidebar |
| `created_at` | timestamptz | `now()` | |
| `updated_at` | timestamptz | `now()` | |

## Indexes

- `ivr_providers_slug_key` — UNIQUE on `slug`
- `ivr_providers_api_key_key` — UNIQUE on `api_key`
- `idx_ivr_providers_active_order` — `(is_active, display_order)` for sidebar queries

## RLS Policies

| Policy | Role | Operations |
|--------|------|-----------|
| `Authenticated users can read providers` | `authenticated` | SELECT |
| `Service role full access` | `service_role` | ALL |

## Realtime

Published via `supabase_realtime` publication. Frontend subscribes via `postgres_changes` for live sidebar/tab updates when providers are added/removed.

## field_mapping — How It Works

Maps dot-notation paths in incoming webhook JSON to `ivr_calls` columns.

**Example stored value:**
```json
{
  "mobile_number": "data.caller",
  "customer_name": "data.callerName",
  "status": "data.status",
  "call_id": "data.callId",
  "did_number": "data.did"
}
```

**Incoming webhook payload:**
```json
{
  "data": {
    "caller": "9876543210",
    "callerName": "Test Farmer",
    "status": "WAITING",
    "callId": "CALL-001",
    "did": "1800123456"
  }
}
```

**Result written to `ivr_calls`:**
```
mobile_number = "9876543210"
customer_name = "Test Farmer"
status = "waiting"          ← after status_map
call_id = "CALL-001"
did_number = "1800123456"
department = "general"      ← from URL slug
```

The backend uses a `resolve_path()` function that traverses nested objects via dot notation (e.g. `data.caller.phone` resolves `obj.data.caller.phone`).

## status_map — How It Works

Translates provider-specific status strings to our canonical statuses.

**Example stored value:**
```json
{
  "ANSWERED": "active",
  "NO_ANSWER": "missed",
  "HANGUP": "hangup",
  "RINGING": "ringing",
  "WAITING": "waiting",
  "COMPLETED": "completed",
  "BUSY": "missed"
}
```

**Canonical statuses:** `waiting`, `ringing`, `active`, `completed`, `missed`, `hangup`

If the incoming status isn't in the map, the raw value is stored as-is.

## Queried By

| Consumer | Filter | Purpose |
|----------|--------|---------|
| `useIVRProviders` (frontend hook) | `is_active = true ORDER BY display_order` | Sidebar items, department tabs, label lookups |
| `useIVRProviders` (Masters page) | `ORDER BY display_order` (all, including inactive) | Admin CRUD |
| `ivrWebhooks.service.ts` (backend) | `slug = ? AND api_key = ?` | Webhook authentication + field mapping |

## Relationship to ivr_calls

```
ivr_providers.slug  ←→  ivr_calls.department
```

Not a foreign key — soft join. The webhook module writes `department = provider.slug` when inserting into `ivr_calls`. This means:
- Adding a new provider immediately creates a new "department" namespace
- All existing IVR hooks (`useIVRCalls`, `useIVRSidebarCounts`) automatically pick up the new department
- No schema changes needed to add new IVR sources

## Current Data

| Slug | Name | Icon | Order | Active |
|------|------|------|-------|--------|
| general | General IVR | Phone | 1 | Yes |
| retailer | Retailer IVR | Users | 2 | Yes |
| app | App IVR | Smartphone | 3 | Yes |
| kyc | KYC IVR | FileCheck | 4 | Yes |
| international | International IVR | Globe | 5 | Yes |
| manish | Manish IVR | Headphones | 6 | Yes |
