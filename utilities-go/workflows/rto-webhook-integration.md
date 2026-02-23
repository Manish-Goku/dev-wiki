# RTO Webhook Integration — Tracking Crons → CRM

`#active` `#webhook` `#cron` `#cross-project`

---

## Purpose

When order tracking crons detect an order's **first** RTO or RTO-in-transit status, fire a webhook to call CRM's cancel-order API so customer metrics (rto_count, total_order_value, etc.) are updated.

---

## Flow

```
Tracking Cron (every 8 min)
  → Poll courier API (Delhivery/Shiprocket)
  → Map courier status to canonical status
  → Status is "rto" or "rto in transit"?
      │
      └── YES → go TriggerRTOCancelWebhook(order, mappedStatus, orderIDStr)
                    │
                    ├── "Only once" guard: check tracking_timestamps.rto / .rto_in_transit
                    │   └── Already set? → return (skip)
                    │
                    ├── lookupMarketplaceName(order.marketplace) → MongoDB marketplaces collection
                    │
                    ├── Flatten order["customer"] bson.M → customerData map
                    │
                    ├── Build updatedData (for conditions):
                    │     {"tracking_status": mappedStatus, "order_id": orderIDStr}
                    │
                    ├── Build buildingData (for payload):
                    │     {"order_id", "customer" (nested), "total_amt", "source", "status": "rto", "tracking_status"}
                    │
                    └── ExecuteWebhookTrigger("order", "", "order.rto", updatedData, buildingData)
                          └── Webhook system handles conditions → payload → auth → POST to CRM
```

---

## Trigger Points in order_tracking.go

| # | Function | Context |
|---|----------|---------|
| 1 | TrackDelhiveryAndUpdateOrders | B2C Delhivery tracking |
| 2 | (same function, primary source path) | B2C Delhivery primary |
| 3 | TrackDelhiveryB2BAndUpdateOrders | B2B Delhivery tracking |
| 4 | TrackShiprocketAndUpdateOrders | Shiprocket tracking |
| 5 | TrackDelhiveryFallbackOrders | Fallback non-primary source |

All use: `go TriggerRTOCancelWebhook(order, mappedStatus, orderIDStr)` — fire-and-forget goroutine.

---

## "Only Once" Guard

**File:** `services/rto_webhook.go`

Before triggering, checks `order["tracking_timestamps"]`:
- If `.rto` exists and is non-nil → skip (already in RTO state)
- If `.rto_in_transit` exists and is non-nil → skip
- If neither set → first RTO detection → proceed with webhook

This prevents duplicate webhook fires if the tracking cron processes the same order multiple times after it enters RTO state.

---

## Marketplace Lookup

```go
lookupMarketplaceName(order["marketplace"])
```

- Takes marketplace ObjectID from order document
- Queries MongoDB `marketplaces` collection by `_id`
- Returns `name` field (e.g., "Shopify", "WooCommerce")
- Returns empty string on failure (5 second timeout)

Used as the `source` field in `buildingData` for payload construction.

---

## Required Webhook Configuration (via API)

To make this work, the following must be configured via the webhook API:

1. **Event:** Create event `order.rto`
2. **Auth:** Create auth with CRM token-based config:
   - `auth_url`: CRM auth endpoint
   - `auth_payload`: `{"client_id": "...", "client_secret": "...", "grant_type": "client_credentials"}`
   - `response_token_path`: path to token in response (e.g., `"access_token"`)
3. **Payload:** Create payload mapping for module `"order"`:
   ```json
   {
     "phone_number": "customer.phone",
     "order_id": "order_id",
     "order_source": "source",
     "order_value": "total_amt",
     "status": "status"
   }
   ```
4. **Condition:** `tracking_status` operator `in` value `rto,rto in transit` (group 1)
5. **Webhook:** Create with:
   - `url`: CRM cancel-order endpoint
   - `module_name`: `"order"`
   - `events`: `[order.rto]`
   - `auth_id`: from step 2
   - `payload_id`: from step 3

---

## Cross-Project Changes (Feb 2026)

| Project | Change |
|---------|--------|
| CRM-Backend | cancel-order route changed PATCH → POST |
| Inventory-Management-Backend | customerHelper.js axios.patch → axios.post |
| utilities-go | Added `status` field to order JSON schema, removed restrictive enums from `tracking_status`, `source`, `courier_name` |

---

## Files

| File | Purpose |
|------|---------|
| `services/rto_webhook.go` | TriggerRTOCancelWebhook, lookupMarketplaceName (97 lines) |
| `services/webhook_trigger.go` | ExecuteWebhookTrigger (441 lines) |
| `services/order_tracking.go` | 5 trigger points in tracking functions |
