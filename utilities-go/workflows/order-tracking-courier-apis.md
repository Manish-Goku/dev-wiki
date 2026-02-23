# Order Tracking — Courier API Details

`#active` `#cron`

---

## Delhivery B2C (Batch)

**Function:** `TrackDelhiveryAndUpdateOrders()`

### API
- **Endpoint:** `DELHIVERY_URL` env var
- **Method:** GET
- **Query param:** `waybill=AWB1,AWB2,...` (comma-separated, max 50)
- **Auth header:** `DELHIVERY_AUTH_TOKEN_B2C`
- **Rate limit:** 750 req/5min (uses 700 to stay safe)

### Query Filter
```
order_status: {$in: [4,5,6,7,8,9,11,14,16,51,26,25]}
isBooked.timestamp: {$gte: now - 31 days}
bookingCourier: /Delhivery/i
source: {$in: ["shopify","woocommerce","Outbound","other","Retailer"]}
awb_number: {$exists: true, $nin: ["", null]}
delhivery_lr_number: absent/null/empty  (excludes B2B)
```
Sort: `last_tracking_time: 1` | Max: 36,000 orders

### Response Format
```json
{
  "ShipmentData": [{
    "Shipment": {
      "AWB": "123456",
      "Status": {
        "Status": "In Transit",
        "StatusType": "UD"
      },
      "Scans": [...],
      "ReferenceNo": "ORDER-001",
      "PickUpDate": "2026-02-20T10:00:00",
      "DeliveryDate": "2026-02-22T14:30:00",
      "RTOStartedDate": "..."
    }
  }]
}
```

### Status Mapping
Uses `MapDelhiveryB2CStatusWithType(status, statusType)` — StatusType-aware:
- **UD** (forward): Manifested→skip, Not Picked→booking_cancelled, In Transit→in transit, Pending→pending/in transit, Dispatched→out_for_delivery
- **DL** (final): Delivered→delivered, RTO→rto
- **RT** (return): In Transit/Pending/Dispatched→rto in transit

### Processing
- 50 worker goroutines process shipments in parallel via channel
- Reference matching: `ReferenceNo` vs `order_id` (handles rebooking suffixes: -C, -C1, -C2, -k, -r)
- `courier_status` format: `{statustype}-{raw_status}` lowercase (e.g., "ud-in_transit")

---

## Delhivery B2B (Single LR)

**Function:** `TrackDelhiveryB2BAndUpdateOrders()`

### API
- **Endpoint:** `https://ltl-clients-api.delhivery.com/lrn/track`
- **Method:** GET
- **Query param:** `lrnum=LR_NUMBER` (one at a time)
- **Auth:** Bearer JWT token
- **Token endpoint:** `https://ltl-clients-api.delhivery.com/ums/login` (POST)
- **Rate limit:** 500 req/5min (uses 480)

### Token Management
Thread-safe `B2BTokenManager`:
- Fetches JWT at start of tracking run
- On 401: auto-refresh with 30-second minimum interval between refreshes
- Prevents multiple simultaneous refresh attempts

### Query Filter
Same as B2C but:
- `delhivery_lr_number: {$exists: true, $nin: ["", null]}` (required for B2B)
- No `awb_number` requirement

### Response Format
```json
{
  "success": true,
  "data": {
    "lrnum": "LR123",
    "status": "In Transit",
    "mcount": 1,
    "wbns": [{
      "status": "In Transit",
      "location": "Delhi Hub",
      "wbn": "AWB456",
      "pickup_date": "2026-02-20",
      "scan_timestamp": "2026-02-21T08:00:00"
    }]
  }
}
```

### Processing
- 3 worker goroutines (configurable via `B2B_TRACKING_WORKERS`)
- Status mapping: `MapDelhiveryB2BStatus()` — same mapping as primary but `noChange=false` for ALL statuses
- Same-day delivery Slack notification for B2B orders

---

## Shiprocket (Batch)

**Function:** `TrackShiprocketAndUpdateOrders()`

### API
- **Endpoint:** `https://apiv2.shiprocket.in/v1/external/courier/track/awbs`
- **Method:** POST
- **Body:** `{"awbs": ["AWB1","AWB2",...]}`
- **Auth:** Bearer token via `GetShiprocketToken()`
- **Batch:** 50 AWBs per request

### Query Filter
Same as B2C but `bookingCourier: /Shiprocket/i`

### Response Format
```json
{
  "AWB1": {
    "tracking_data": {
      "track_status": 1,
      "shipment_status": 7,
      "shipment_track": [{
        "awb_code": "AWB1",
        "current_status": "DELIVERED",
        "courier_name": "Delhivery",
        "updated_time_stamp": "2026-02-22 14:30:00"
      }],
      "shipment_track_activities": [{
        "date": "2026-02-22 14:30:00",
        "status": "Delivered",
        "sr-status-label": "DELIVERED"
      }]
    }
  }
}
```

### Status Extraction Priority
1. `shipment_track[0].current_status`
2. Fallback: `shipment_track_activities[0].sr-status-label`

### Processing
- 50 worker goroutines for shipment processing
- On 401: refresh token and retry batch
- Status mapping: `MapShiprocketStatus()`

---

## Velocity (Batch)

**Function:** `TrackVelocityAndUpdateOrders()`

### API
- **Auth endpoint:** `https://shazam.velocity.in/custom/api/v1/auth-token` (POST)
  - Body: `{"username": "...", "password": "..."}`
  - Response: `{"token": "...", "expires_at": "..."}`
- **Tracking endpoint:** `https://shazam.velocity.in/custom/api/v1/order-tracking` (POST)
  - Body: `{"awbs": ["AWB1","AWB2",...]}`
  - Auth: Bearer token
- **Batch:** 45 AWBs (50 limit, using 45 for safety)

### Query Filter
Same as B2C but `bookingCourier: /velocity/i`

### Response Format
```json
{
  "result": {
    "AWB1": {
      "tracking_data": {
        "shipment_status": "IN_TRANSIT",
        "shipment_track": [{
          "current_status": "IN_TRANSIT",
          "pickup_date": "2026-02-20",
          "delivered_date": ""
        }],
        "shipment_track_activities": [{
          "date": "2026-02-21 08:00:00",
          "location": "Delhi Warehouse"
        }]
      }
    }
  }
}
```

### Status Extraction Priority
1. `shipment_status` field
2. Fallback: `shipment_track[0].current_status`

### Processing
- Status mapping: `MapVelocityStatus()`
- Env vars: `VELOCITY_USERNAME`, `VELOCITY_PASSWORD`

---

## Delhivery Fallback (Single AWB)

**Function:** `TrackDelhiveryFallbackOrders()`

### API
- **Endpoint:** `DELHIVERY_FALLBACK_URL` env var
- **Method:** GET
- **Query param:** `wbn=AWB` (note: `wbn` not `waybill`)
- **Auth:** Same as B2C token
- **Timeout:** 30 seconds per request

### Query Filter
Same base but:
- `source: {$nin: PrimarySourcesList}` (non-primary sources only)
- `being_tracked: true OR missing/null` (excludes explicitly stopped orders)

### End State Detection
Special response messages stop future tracking:
- "tracking details are not visible if the shipment reached end state 7 or more days ago" → `being_tracked = false`
- "invalid AWB or very old package" → `being_tracked = false`

### Processing
- 3 workers (configurable via `FALLBACK_TRACKING_WORKERS`)
- **5-second delay** between API calls per worker (avoids rate limiting)
- Status mapping: `MapDelhiveryFallbackStatus()` (same as primary mapping)

---

## Rate Limiting (All Trackers)

Each tracker has independent global state:
```go
var {tracker}RateLimitMu sync.RWMutex
var {tracker}RateLimitResumeAt time.Time
```

**On 429:** Parse `Retry-After` header (7+ header name variants, seconds/unix/RFC1123 formats), default 5 min wait.
**On 403:** 8-minute wait.
**On next cron:** `is{Tracker}RateLimited()` → skip entire run if still limited.
**Context cancellation:** On rate limit hit, cancel all in-flight workers.

---

## Email Reports

Each tracker sends an SMTP email summary after completion:
- Sender: `EMAIL_SENDER` env var (Gmail)
- Password: `EMAIL_PASSWORD` env var
- Recipients: `EMAIL_RECIPIENTS` env var (comma-separated)
- Content: HTML table with tracking results
