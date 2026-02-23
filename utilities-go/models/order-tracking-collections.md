# Order Tracking — MongoDB Collections

`#active` `#cron`

---

## orders Collection (Read + Write)

The primary order document. Tracking crons update these fields:

### Fields Updated by Tracking

| Field | Type | Update Behavior |
|-------|------|-----------------|
| `last_tracking_time` | Date | Always updated to current time (even if status unchanged) |
| `order_status` | Number | Status ID from `status` collection (only if status changed) |
| `courier_status` | String | Raw courier status (B2C: `{statustype}-{raw_status}`) |
| `tracking_timestamps.<key>` | Date | First occurrence only — `$set` with dot notation, preserves original |
| `order_status_map` | Array[ObjectID] | `$push` status ObjectID only if different from last entry |

### Fields Read by Tracking

| Field | Type | Used For |
|-------|------|----------|
| `_id` | ObjectID | Document identifier |
| `order_id` | String | Order ID for matching, logging, webhook payload |
| `awb_number` | String | AWB for B2C/Shiprocket/Velocity/Fallback API calls |
| `delhivery_lr_number` | String | LR number for B2B API calls |
| `bookingCourier` | String | Determines which tracker handles the order |
| `source` | String | Primary vs fallback classification |
| `order_status` | Number | Current status (filter + comparison) |
| `isBooked.timestamp` | Date | Buffer period filter (last 31 days) |
| `last_tracking_time` | Date | Sort by oldest tracked first |
| `being_tracked` | Boolean | Fallback: false = stop tracking |
| `marketplace` | ObjectID | RTO webhook: lookup marketplace name |
| `customer` | Object | RTO webhook: flatten for payload building |
| `total_amt` | Number | RTO webhook: payload field |
| `tracking_timestamps` | Object | RTO webhook: "only once" guard |
| `order_status_map` | Array | Check last status to avoid duplicate push |

### Tracking Query Filters

**Base filter (all trackers):**
```javascript
{
  order_status: {$in: [4,5,6,7,8,9,11,14,16,51,26,25]},
  "isBooked.timestamp": {$gte: ISODate("31 days ago")}
}
```

**B2C additions:**
```javascript
{
  bookingCourier: /Delhivery/i,
  source: {$in: ["shopify","woocommerce","Outbound","other","Retailer"]},
  awb_number: {$exists: true, $nin: ["", null]},
  $or: [
    {delhivery_lr_number: {$exists: false}},
    {delhivery_lr_number: null},
    {delhivery_lr_number: ""}
  ]
}
```

**B2B additions:**
```javascript
{
  bookingCourier: /Delhivery/i,
  source: {$in: ["shopify","woocommerce","Outbound","other","Retailer"]},
  delhivery_lr_number: {$exists: true, $nin: ["", null]}
}
```

**Shiprocket additions:** Same as B2C but `bookingCourier: /Shiprocket/i`
**Velocity additions:** Same as B2C but `bookingCourier: /velocity/i`
**Fallback additions:** `source: {$nin: PrimarySourcesList}`, `being_tracked != false`

---

## orderTracking Collection (Read + Write)

Maintains a separate tracking history per order.

### Schema

| Field | Type | Description |
|-------|------|-------------|
| `orderId` | String | Order ID (indexed, used for upsert) |
| `currentStatus` | String | Current canonical status |
| `stockShipStatus` | Number | Status ID from status collection |
| `History` | Array | Status change history |
| `History[].status` | String | Canonical status at that time |
| `History[].timeStamp` | Date | When status was recorded |

### Update Pattern — `UpdateOrderTrackingWithHistory()`

**File:** `utils/tracking_helper.go`

Two-step atomic approach to prevent duplicate History entries:

**Step 1: Upsert**
```javascript
// If document doesn't exist, create with first History entry
// If exists, only update $set fields (no push)
db.orderTracking.updateOne(
  {orderId: "ORDER-001"},
  {
    $set: {currentStatus: "in transit", stockShipStatus: 5},
    $setOnInsert: {History: [{status: "in transit", timeStamp: ISODate()}]}
  },
  {upsert: true}
)
```

**Step 2: Conditional Push (only if last status differs)**
```javascript
// Atomically check last History entry's status
// Only push if different from new status
db.orderTracking.updateOne(
  {
    orderId: "ORDER-001",
    $or: [
      {History: {$size: 0}},
      {$expr: {$ne: [
        {$let: {vars: {last: {$arrayElemAt: ["$History", -1]}}, in: "$$last.status"}},
        "in transit"
      ]}}
    ]
  },
  {$push: {History: {status: "in transit", timeStamp: ISODate()}}}
)
```

This prevents race conditions from concurrent tracking workers.

---

## status Collection (Read-only)

Maps canonical status names to numeric IDs.

### Usage
```go
var statusDoc bson.M
statusColl.FindOne(ctx, bson.M{"status": "in transit"}).Decode(&statusDoc)
statusID := statusDoc["status_id"]  // e.g., 5
```

Used to populate `orders.order_status` and `orderTracking.stockShipStatus`.

---

## marketplaces Collection (Read-only)

Queried by RTO webhook to resolve marketplace name.

```go
coll.FindOne(ctx, bson.M{"_id": marketplaceObjectID}).Decode(&result)
name := result["name"]  // e.g., "Shopify"
```

---

## Update Transaction Pattern

```go
// Atomic order update
updateSet := bson.M{
    "last_tracking_time": now,
}
if statusChanged {
    updateSet["order_status"] = statusID
    updateSet["courier_status"] = courierStatus
    updateSet["tracking_timestamps." + statusKey] = now  // only if not exists
}

updateOp := bson.M{"$set": updateSet}
if shouldPushStatus {
    updateOp["$push"] = bson.M{"order_status_map": statusObjectID}
}

ordersColl.UpdateByID(ctx, orderOID, updateOp)

// Then update orderTracking
UpdateOrderTrackingWithHistory(orderIDStr, canonicalStatus, statusID)
```
