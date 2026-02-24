# Dynamic IVR Providers — Workflow

End-to-end flow for dynamically registering IVR providers, receiving webhook calls, and auto-rendering them in the frontend. `#active` `#webhook`

## Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  ADMIN (Masters Page)                                           │
│  1. Creates provider: name, slug, icon, field_mapping, status_map│
│  2. Gets: webhook URL + API key                                  │
│  3. Configures IVR platform to POST to webhook URL               │
└─────────────────┬───────────────────────────────────────────────┘
                  │ writes to
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  SUPABASE: ivr_providers                                        │
│  slug, name, api_key, field_mapping, status_map, icon, ...      │
└────────┬──────────────────────────────────┬─────────────────────┘
         │ postgres_changes (realtime)      │ queried by backend
         ▼                                  ▼
┌──────────────────────┐    ┌──────────────────────────────────────┐
│  FRONTEND            │    │  BACKEND: POST /webhooks/ivr/:slug   │
│  useIVRProviders()   │    │  1. Lookup provider by slug          │
│  - Sidebar items     │    │  2. Validate x-api-key header        │
│  - Department tabs   │    │  3. Return 200 immediately           │
│  - Badge counts      │    │  4. Async: apply field_mapping       │
│  - Masters CRUD      │    │  5. Async: apply status_map          │
└──────────────────────┘    │  6. Async: upsert ivr_calls          │
                            └────────────────┬─────────────────────┘
                                             │ writes to
                                             ▼
                            ┌──────────────────────────────────────┐
                            │  SUPABASE: ivr_calls                 │
                            │  department = slug                   │
                            │  mobile_number, status, call_id, ... │
                            └────────────────┬─────────────────────┘
                                             │ postgres_changes
                                             ▼
                            ┌──────────────────────────────────────┐
                            │  FRONTEND: IVR Pages                 │
                            │  useIVRCalls({ department: slug })   │
                            │  - Live calls, hangups, call history │
                            │  - Real-time updates                 │
                            └──────────────────────────────────────┘
```

## Step 1: Register a Provider (Admin)

**Where:** Masters page → IVR Providers tab (`/masters`)

**Actions available:**
- **Add Provider**: Name, Slug (auto-generated from name), Icon (picker), Display order, DID numbers, Field Mapping rows, Status Map rows
- **Edit Provider**: All fields except slug (immutable after creation)
- **Deactivate**: Soft-delete (`is_active = false`) — hides from sidebar, stops processing webhooks
- **Regenerate API Key**: Invalidates old key immediately, generates new UUID
- **Copy Webhook URL**: `{BACKEND_URL}/webhooks/ivr/{slug}`
- **Copy API Key**: Masked by default, toggle visibility

**On create/update:** Supabase realtime fires `postgres_changes` → all connected frontends update sidebar/tabs instantly.

## Step 2: Configure IVR Platform

Give the IVR platform (Knowlarity, Exotel, Ozonetel, etc.) the webhook URL and API key.

**Webhook URL format:** `POST https://{backend-host}/webhooks/ivr/{slug}`

**Required headers:**
```
x-api-key: {api_key from ivr_providers}
Content-Type: application/json
```

**Body:** Any JSON. The `field_mapping` config tells the backend where to find each field.

## Step 3: Webhook Processing (Backend)

**Module:** `src/ivrWebhooks/` (NestJS)

**Controller:** `ivrWebhooks.controller.ts`
```
POST /webhooks/ivr/:slug
Headers: x-api-key: {key}
Body: { ...any JSON from IVR platform }
Response: 200 { status: "ok" }  ← returned IMMEDIATELY (fire-and-forget)
```

**Service:** `ivrWebhooks.service.ts` — runs async after 200 response:

1. **Lookup provider**: `SELECT * FROM ivr_providers WHERE slug = :slug`
2. **Validate API key**: Compare `x-api-key` header with `provider.api_key` → reject if mismatch (logged, not returned to caller since 200 already sent)
3. **Apply field_mapping**: For each key-value pair in `field_mapping`:
   - Key = our `ivr_calls` column name (e.g. `mobile_number`)
   - Value = dot-notation path into webhook body (e.g. `data.caller`)
   - `resolve_path("data.caller", body)` → traverses `body.data.caller`
4. **Apply status_map**: If `status` was mapped, look up the raw value in `status_map` → replace with canonical status
5. **Generate call_id**: If `call_id` wasn't mapped, auto-generate as `{SLUG}-{timestamp}`
6. **Upsert**: Insert into `ivr_calls` with `department = slug`. On conflict (`call_id`), update status + timestamp fields.

**Error handling:** All errors caught and logged via NestJS Logger. Since 200 is already returned, webhook caller never sees errors. Check backend logs for debugging.

### resolve_path() — Dot-Notation Traversal

```typescript
// resolve_path("data.caller.phone", { data: { caller: { phone: "9876543210" } } })
// → "9876543210"

function resolve_path(path: string, obj: Record<string, unknown>): unknown {
  return path.split('.').reduce((acc, key) => acc?.[key], obj);
}
```

## Step 4: Frontend Auto-Rendering

**No code changes needed.** Everything is data-driven from `ivr_providers` table.

### Sidebar (`AppSidebar.tsx`)
```
useIVRProviders() → providers[]
  ↓
IVR Live section:
  - {provider.name} → /ivr-live/{provider.slug}  (badge: ivrCounts.live[slug])
  - All IVR Live (Admin) → /ivr-live/admin

IVR Hangup section:
  - {provider.name} Hangup → /ivr-hangup/{provider.slug}  (badge: ivrCounts.hangup[slug])
```

### Department Tabs (`DepartmentSubTabs.tsx`)
```
useIVRDepartmentTabs() → { live: tabs[], hangup: tabs[] }
  ↓
Tabs: [All, ...providers.map(p => { key: p.slug, label: p.name })]
```

### Sidebar Counts (`useIVRSidebarCounts.ts`)
```
SELECT department, status, is_sla_breached FROM ivr_calls
  ↓
Group by department × status → Record<string, number>
  live[dept] = count where status IN ('waiting', 'ringing', 'missed')
  hangup[dept] = count where status = 'hangup'
```

Dynamic `Record<string, number>` — any new department automatically gets counted.

### Department Pages (`IVRLiveDepartment.tsx`, `IVRHangupDepartment.tsx`)
```
URL: /ivr-live/:department
  ↓
useIVRCalls({ department: params.department, statuses: [...] })
useIVRProviders().getLabel(department)  → page title
useCallCategories(department)  → disposition dropdowns
```

## Step 5: Adding a New IVR Provider (Zero-Code Example)

1. Admin opens Masters → IVR Providers → "Add Provider"
2. Fills: Name="Agronomy IVR", Slug auto-fills "agronomy", Icon="Leaf", Order=7
3. Configures field mapping (which webhook JSON fields map to our columns)
4. Configures status mapping (provider's status strings → our canonical statuses)
5. Clicks "Create Provider"
6. **Instantly**: Sidebar shows "Agronomy IVR" under IVR Live + IVR Hangup
7. Admin copies webhook URL + API key → configures in IVR platform
8. Calls come in → appear on `/ivr-live/agronomy` page in real-time

**Zero frontend code changes. Zero backend code changes. Zero deployments.**

## Files Involved

### Backend
| File | Role |
|------|------|
| `src/ivrWebhooks/ivrWebhooks.module.ts` | Module registration |
| `src/ivrWebhooks/ivrWebhooks.controller.ts` | `POST /webhooks/ivr/:slug` endpoint |
| `src/ivrWebhooks/ivrWebhooks.service.ts` | Field mapping, status mapping, upsert |
| `src/app.module.ts` | Imports `IvrWebhooksModule` |

### Frontend
| File | Role |
|------|------|
| `src/hooks/useIVRProviders.ts` | Provider data + CRUD + realtime |
| `src/hooks/useCallCategories.ts` | Dynamic categories per department |
| `src/hooks/useIVRSidebarCounts.ts` | Dynamic badge counts |
| `src/lib/iconMap.ts` | Icon name → React component |
| `src/components/masters/IVRProvidersTab.tsx` | Admin CRUD UI |
| `src/components/layout/AppSidebar.tsx` | Dynamic sidebar items |
| `src/components/shared/DepartmentSubTabs.tsx` | Dynamic tab generation |
| `src/integrations/supabase/types.ts` | Table type definition (required!) |

### Database
| Table | Role |
|-------|------|
| `ivr_providers` | Provider registry (slug, field_mapping, status_map, api_key) |
| `ivr_calls` | Call records (department = provider slug) |

## Canonical Statuses

| Status | Meaning | Sidebar group |
|--------|---------|---------------|
| `waiting` | In queue, no agent yet | Live |
| `ringing` | Ringing at agent's phone | Live |
| `active` | Agent is on the call | — (not counted in sidebar) |
| `missed` | No agent answered | Live |
| `hangup` | Customer hung up before answer | Hangup |
| `completed` | Call finished normally | — (historical) |

## Testing

### Quick webhook test
```bash
# Get API key from Masters page or DB
curl -X POST \
  -H "x-api-key: {api_key}" \
  -H "Content-Type: application/json" \
  -d '{"data":{"caller":"9876543210","callerName":"Test","status":"WAITING","callId":"TEST-001"}}' \
  http://localhost:3002/webhooks/ivr/{slug}
```

### Verify call landed
```sql
SELECT call_id, mobile_number, department, status FROM ivr_calls WHERE call_id = 'TEST-001';
```

### Invalid API key test
Same curl with wrong key → returns 200 (fire-and-forget) but no row inserted. Check backend logs for error.

## Gotchas

1. **Supabase types file**: New tables MUST be added to `src/integrations/supabase/types.ts`. The typed client silently returns empty results for unknown tables — no runtime error.
2. **Fire-and-forget**: Webhook always returns 200. Errors are logged server-side only. Debug via backend console or Supabase logs.
3. **Slug immutability**: Changing a slug would orphan existing `ivr_calls` rows. Slug is disabled in edit mode.
4. **API key rotation**: Regenerating a key invalidates the old one immediately. IVR platform must be updated.
5. **Hangup page call categorization**: Category filters in `IVRHangupDepartment.tsx` (lines 74-79) MUST be exhaustive. A call that doesn't match any filter becomes invisible. Fixed 2026-02-23: unassigned calls with attempts > 0 were falling through all filters. Fix: `pendingCalls` now includes all unassigned+unresolved calls regardless of attempt count.
