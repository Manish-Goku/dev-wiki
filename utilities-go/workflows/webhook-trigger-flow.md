# Webhook Trigger Flow — End-to-End Execution

`#active` `#webhook` `#event-driven`

---

## Entry Point

```go
services.ExecuteWebhookTrigger(moduleName, source, event string, updatedData, buildingData map[string]interface{})
```

**File:** `services/webhook_trigger.go`

**Parameters:**
| Param | Purpose | Example |
|-------|---------|---------|
| `moduleName` | Which module's webhooks to fire | `"order"` |
| `source` | Filter by source (empty = no filter) | `"MOBILE"`, `""` |
| `event` | Event name to match | `"order.rto"` |
| `updatedData` | Data for condition evaluation | `{"tracking_status": "rto", "order_id": "123"}` |
| `buildingData` | Data for payload construction | `{"order_id": "123", "customer": {"email": "...", "phone": "..."}, ...}` |

Can be called:
- **Programmatically:** Any Go service imports and calls directly (e.g., `rto_webhook.go`)
- **Via HTTP:** `POST /webhooks/trigger` (manual testing)

---

## Execution Pipeline

### Step 1: Fetch Active Webhooks

```
SELECT * FROM webhooks
WHERE module_name = ? AND is_active = true
PRELOAD: Auth, Payload, Events, Sources, Conditions
```

If no webhooks found → return empty response (not an error).

### Step 2: Filter by Event and Source

```go
filterWebhooksByEventAndSource(webhooks, event, source)
```

- Webhook must have the triggered event in its `Events` list
- If `source` is non-empty AND webhook has `Sources` configured, source must match
- If webhook's `Sources` is empty → accepts any source

### Step 3: Execute Each Matching Webhook

For each filtered webhook, `executeSingleWebhook()` runs:

#### 3a. Evaluate Conditions

```go
EvaluateConditions(webhook.Conditions, updatedData)
```

- Group conditions by `ConditionGroup` field
- **AND** all conditions within the same group
- **OR** across different groups
- If conditions not met → skip webhook (no log created... returns result with "conditions not met")

**Value extraction:** `ExtractValueFromPath(fieldPath, data)` — traverses nested maps using dot notation.

**Type coercion for numeric operators:** `toFloat64()` handles float64, float32, int, int64, and string-to-float parsing.

**Operators:**
| Operator | Behavior |
|----------|----------|
| `==` | `fmt.Sprintf("%v", actual) == value` |
| `!=` | `fmt.Sprintf("%v", actual) != value` |
| `>`, `<`, `>=`, `<=` | Numeric: converts both sides via `toFloat64()` |
| `contains` | `strings.Contains(fmt.Sprintf("%v", actual), value)` |
| `in` | Splits value by `,`, trims spaces, checks if actual matches any |

#### 3b. Build Payload

```go
BuildPayload(webhook.Payload, buildingData, event, webhook.ID)
```

- If no `Payload` config → uses raw `buildingData` as payload
- Unmarshals `PayloadMapping` JSONB → `map[string]string`
- For each `"outKey": "dot.path"`, extracts value from `buildingData` via `ExtractValueFromPath`
- If `IncludeEventMeta = true`, appends:
  ```json
  {"_event_meta": {"webhook_id": 12, "trigger_time": "2026-02-23T10:00:00Z", "event_type": "order.rto"}}
  ```

#### 3c. Validate Payload Against JSON Schema

```go
schemaManager.ValidatePayload(moduleName, payload)
```

- Loads schema from embedded `utils/schemas/<module>.json` (cached in singleton `SchemaManager`)
- Validates types, formats (email, phone), patterns (pincode), ranges, enums
- **Graceful fallback:** If no schema exists for the module, validation passes

#### 3d. Execute HTTP Request

```go
ExecuteWebhook(url, payload, auth)  // services/webhook.go
```

- **Method:** POST (hardcoded)
- **Timeout:** 15 seconds
- **Content-Type:** application/json
- **Auth application:**
  - If `auth.Headers` set → unmarshal JSON, set each header on request
  - Else if `auth.AuthURL` set → `fetchAuthToken()`: POST to AuthURL with AuthPayload, extract token via dot-path (`ResponseTokenPath`), set `Authorization: Bearer <token>`
- **Success criteria:** HTTP 2xx response

#### 3e. Create WebhookLog

On **success:**
```
WebhookLog{Success: true, AttemptCount: 1, SuccessPayloadSnapshot: <payload>, CompletedAt: now}
```

On **failure:**
```
WebhookLog{Success: false, AttemptCount: 1, FailedLogs: [{AttemptNumber: 1, FailureReason: "...", HTTPStatus: 502, PayloadSnapshot: <payload>, TimeTakenMs: 15000}]}
```

### Step 4: Aggregate Results

Returns `TriggerWebhookResponse`:
```json
{
  "triggered_webhooks": 2,
  "failed_webhooks": 1,
  "results": [
    {"webhook_id": 5, "webhook_url": "https://...", "success": true, "time_taken_ms": 250, "log_id": 42},
    {"webhook_id": 8, "webhook_url": "https://...", "success": false, "error": "webhook responded with status 500", "time_taken_ms": 3200, "log_id": 43}
  ]
}
```

---

## OrderID Extraction

`GetOrderIDFromData(data)` priority:
1. `order_id`
2. `id`
3. `shipment_id`
4. `transaction_id`
5. Fallback: `trigger_{unix_timestamp}`

---

## Manual Retry Flow

**Endpoint:** `POST /webhooks/logs/:id/retry`
**File:** `handlers/webhook_handlers/webhook_handler.go`

1. Fetch WebhookLog by ID
2. Fetch parent Webhook with Payload preloaded
3. Verify FailedLogs exist (must have previous failure)
4. **Payload selection:**
   - If request body has `payload` → validate against JSON schema + mapping structure → use custom
   - Otherwise → use last failed attempt's `PayloadSnapshot`
5. Execute webhook with chosen payload
6. Append new `FailedLog` entry (whether success or failure)
7. If success: set `Success=true`, store `SuccessPayloadSnapshot`, set `CompletedAt`
8. Increment `AttemptCount`, save updated log

---

## Flow Diagram

```
ExecuteWebhookTrigger(module, source, event, updatedData, buildingData)
    │
    ├── Query: active webhooks for module (GORM preload all relations)
    │
    ├── Filter: match event + source
    │
    └── For each webhook:
        │
        ├── EvaluateConditions(conditions, updatedData)
        │   ├── Group by condition_group
        │   ├── AND within group
        │   └── OR across groups
        │       └── Not met? → skip (return "conditions not met")
        │
        ├── BuildPayload(payloadConfig, buildingData, event, webhookID)
        │   ├── Map fields via dot-notation
        │   └── Optionally add _event_meta
        │
        ├── ValidatePayload(moduleName, payload)
        │   └── JSON schema check (graceful if no schema)
        │
        ├── ExecuteWebhook(url, payload, auth)
        │   ├── Apply auth (headers or OAuth2 token)
        │   └── HTTP POST, 15s timeout, expect 2xx
        │
        └── Create WebhookLog (success or failure details)
```

---

## Important Notes

- **No automatic retry** — `RetryEnabled`/`MaxRetries` fields exist but auto-retry on failure is NOT implemented. Retry is manual only via the API endpoint.
- **Synchronous execution** — Webhooks in a single trigger call execute sequentially (not in parallel goroutines).
- **Fire-and-forget from callers** — `TriggerRTOCancelWebhook` is called with `go` keyword from tracking crons, making it non-blocking to the tracking loop.
