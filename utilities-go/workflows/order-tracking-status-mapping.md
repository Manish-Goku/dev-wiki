# Order Tracking — Status Mapping Reference

`#active` `#cron`

---

## Canonical Statuses

These are the normalized statuses stored in MongoDB across all couriers:

| Canonical Status | Description | Side Effects |
|-----------------|-------------|--------------|
| pending | Order booked, not yet picked | — |
| in transit | Picked up, moving to destination | — |
| out_for_delivery | On delivery vehicle | — |
| delivered | Successfully delivered | Slack (B2B same-day) |
| ndr | Non-Delivery Report / failed attempt | — |
| rto | Return-To-Origin initiated | RTO Webhook → CRM |
| rto in transit | RTO shipment in transit back | RTO Webhook → CRM |
| booking_cancelled | Not picked up / cancelled | Slack notification |
| lost | Shipment lost | Slack notification |
| destroy | Shipment destroyed | Slack notification |

---

## Delhivery B2C — StatusType Mapping

**File:** `utils/delhivery_status_mapping.go`
**Function:** `MapDelhiveryB2CStatusWithType(status, statusType string)`

Uses the `StatusType` field (UD/DL/RT) for directional awareness:

### StatusType = "UD" (Undelivered / Forward Journey)

| Raw Status | Canonical | Notes |
|-----------|-----------|-------|
| Manifested | *(no change)* | noChange=true, skip update |
| Not Picked | booking_cancelled | |
| In Transit | in transit | |
| Pending | pending OR in transit | If PickUpDate < today → in transit, else pending |
| Dispatched | out_for_delivery | |

### StatusType = "DL" (Delivered / Final)

| Raw Status | Canonical |
|-----------|-----------|
| Delivered | delivered |
| RTO | rto |

### StatusType = "RT" (Return Journey)

| Raw Status | Canonical |
|-----------|-----------|
| In Transit | rto in transit |
| Pending | rto in transit |
| Dispatched | rto in transit |

**Normalization:** Before mapping, statuses are normalized — uppercase, spaces/hyphens → underscores, special chars removed.

**courier_status format:** `{statustype}-{raw_status}` lowercase (e.g., "ud-in_transit", "dl-delivered", "rt-pending")

---

## Delhivery Primary/B2B/Fallback — Legacy Mapping

**Function:** `MapDelhiveryPrimaryStatus()` / `MapDelhiveryB2BStatus()` / `MapDelhiveryFallbackStatus()`

All use the same `DelhiveryPrimaryStatusMapping` table (80+ entries). Key mappings:

### No Change (noChange=true) — B2C Primary only
| Raw Status (normalized) |
|------------------------|
| PICK_UP_PENDING, MANIFESTED, WAITING_PICKUP, PICKUP_QUEUED |
| PICKUP_RESCHEDULED, PICK_REQUESTED, SHIPMENT_BOOKED |

### in transit
| Raw Status (normalized) |
|------------------------|
| IN_TRANSIT, PICKED_UP, DISPATCHED, RECEIVED_AT_ORIGIN |
| REACHED_AT_DESTINATION, BAGGED, RECEIVED, ADDED_TO_BAG |
| CONNECTION_ALLOCATED, IN_TRANSIT_TO, PICKUP_COMPLETE |
| SHIPPED, IN_TRANSIT_TO_NEXT_FACILITY |

### out_for_delivery
| Raw Status (normalized) |
|------------------------|
| OFD, OUT_FOR_DELIVERY |

### delivered
| Raw Status (normalized) |
|------------------------|
| DELIVERED, SHIPMENT_DELIVERED |

### rto
| Raw Status (normalized) |
|------------------------|
| RTO, RETURNED, OUT_FOR_RETURN, RTO_DELIVERED |
| RETURNED_TO_ORIGIN, RTO_COMPLETED |

### rto in transit
| Raw Status (normalized) |
|------------------------|
| RETURNED_INTRANSIT, RTO_IN_TRANSIT, RTO_IN_INTRANSIT |
| RTO_OUT_FOR_DELIVERY, RTO_OFD |

### booking_cancelled
| Raw Status (normalized) |
|------------------------|
| NOT_PICKED, CANCELLED, BOOKING_CANCELLED |

### ndr
| Raw Status (normalized) |
|------------------------|
| NDR, FAILED_DELIVERY, UNDELIVERED |

### lost
| Raw Status (normalized) |
|------------------------|
| LOST, MISSING, UNTRACEABLE |

### destroy
| Raw Status (normalized) |
|------------------------|
| DESTROYED, DISPOSED_OFF, DAMAGED |

**B2B difference:** `MapDelhiveryB2BStatus` returns `noChange=false` for ALL statuses, meaning even unmapped statuses update the database.

---

## Shiprocket Status Mapping

**File:** `utils/shiprocket_status_mapping.go`
**Function:** `MapShiprocketStatus(status string)`

### No Change (noChange=true)
| Raw Status |
|-----------|
| SHIPPED, PICKUP_RESCHEDULED, OUT_FOR_PICKUP, PENDING |
| PICKUP_ERROR, AWB_ASSIGNED, LABEL_GENERATED |
| PICKUP_SCHEDULED, PICKUP_QUEUED |

### in transit
| Raw Status |
|-----------|
| IN_TRANSIT, PICKED_UP, REACHED_DEST_CITY |
| REACHED_AT_DESTINATION, RECEIVED_AT_ORIGIN |
| IN_TRANSIT_TO_NEXT_FACILITY, DISPATCHED |

### out_for_delivery
| Raw Status |
|-----------|
| OUT_FOR_DELIVERY |

### delivered
| Raw Status |
|-----------|
| DELIVERED |

### ndr
| Raw Status |
|-----------|
| NDR, FAILED_DELIVERY |

### rto
| Raw Status |
|-----------|
| RTO, RTO_INITIATED, RTO_DELIVERED |

### rto in transit
| Raw Status |
|-----------|
| RTO_IN_TRANSIT, RTO_OFD, RTO_OUT_FOR_DELIVERY |

### booking_cancelled
| Raw Status |
|-----------|
| CANCELLED, NOT_PICKED, CANCELED_BY_SELLER |

### lost
| Raw Status |
|-----------|
| LOST, UNTRACEABLE, MISSING |

### destroy
| Raw Status |
|-----------|
| DESTROYED, DISPOSED_OFF |

**Status extraction priority:**
1. `shipment_track[0].current_status`
2. `shipment_track_activities[0].sr-status-label`

---

## Velocity Status Mapping

**File:** `utils/velocity_status_mapping.go`
**Function:** `MapVelocityStatus(status string)`

| Raw Status | Canonical |
|-----------|-----------|
| CANCELLED, NOT_PICKED | booking_cancelled |
| IN_TRANSIT, SHIPPED, PICKED_UP | in transit |
| MANIFEST_UPLOADED | *(no change)* |
| OUT_FOR_DELIVERY | out_for_delivery |
| DELIVERED | delivered |
| RTO, RTO_INITIATED | rto |
| RTO_IN_TRANSIT, RETURN_TO_ORIGIN | rto in transit |
| NDR, UNDELIVERED, FAILED_ATTEMPT | ndr |
| LOST, MISSING | lost |
| DAMAGED, DESTROYED | destroy |

**Status extraction priority:**
1. `shipment_status` field
2. `shipment_track[0].current_status`

---

## tracking_timestamps Field Structure

Stored in each order document. Records the **first** time each canonical status was reached.

```json
{
  "tracking_timestamps": {
    "in_transit": "2026-02-20T10:00:00Z",
    "out_for_delivery": "2026-02-22T08:30:00Z",
    "delivered": "2026-02-22T14:15:00Z"
  }
}
```

- Key: canonical status (lowercase, spaces → underscores)
- Value: ISO timestamp of first occurrence
- Only set via `$set` if field doesn't already exist (preserves original timestamp)
- Used by RTO webhook "only once" guard to prevent duplicate triggers

---

## Unmapped Status Handling

If a raw courier status doesn't match any mapping entry:
- Logged as alert (⚠️ unmapped status) to tracking logger
- `last_tracking_time` still updated (prevents re-processing starvation)
- No order_status or tracking_timestamps change
- B2B exception: unmapped statuses still attempt update (noChange always false)
