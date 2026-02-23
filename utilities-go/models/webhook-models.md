# Webhook Models — PostgreSQL (GORM)

All webhook configuration lives in PostgreSQL, managed via GORM AutoMigrate at startup (`database/postgress.go`).

`#active` `#webhook`

---

## Webhook

**Table:** `webhooks`
**File:** `models/webhook_models/webhook.go`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| ID | uint | PK | Auto-increment |
| URL | string | NOT NULL | Endpoint to POST to |
| AuthID | *uint | FK → webhook_auths | Optional auth config |
| PayloadID | *uint | FK → webhook_payloads | Optional payload mapping |
| RegisteredBy | string | NOT NULL | Creator email/username |
| Description | string | — | Human-readable purpose |
| IsActive | bool | NOT NULL, default: true | Enable/disable |
| ModuleName | string | NOT NULL | Subscribing module (e.g., "order") |
| RetryEnabled | bool | NOT NULL, default: true | Enable retry (not auto-implemented yet) |
| MaxRetries | int | NOT NULL, CHECK >= 0 | Max retry attempts |
| CreatedAt | time.Time | Auto | — |
| UpdatedAt | time.Time | Auto | — |

**Relationships:**
- `Auth` → WebhookAuth (belongs-to, optional)
- `Payload` → WebhookPayload (belongs-to, optional)
- `Events` → []WebhookEvent (many-to-many via `webhook_subscribed_events`)
- `Sources` → []WebhookSource (many-to-many via `webhook_allowed_sources`)
- `Conditions` → []WebhookCondition (has-many)

---

## WebhookAuth

**Table:** `webhook_auths`
**File:** `models/webhook_models/webhook_auth.go`

| Field | Type | Description |
|-------|------|-------------|
| ID | uint | PK |
| Headers | JSONB | Static headers (key-value pairs) |
| AuthURL | string | Token endpoint URL |
| AuthPayload | JSONB | Payload sent to AuthURL |
| ResponseTokenPath | string | Dot-path to extract token from response (e.g., "data.access_token") |
| CreatedAt | time.Time | Auto |

**Two mutually exclusive modes:**
1. **Static Headers** — Set `Headers` only. Headers injected directly into webhook request.
2. **Token-based (OAuth2)** — Set `AuthURL` + `AuthPayload` + `ResponseTokenPath`. System POSTs to AuthURL, extracts token, injects as `Authorization: Bearer <token>`.

---

## WebhookPayload

**Table:** `webhook_payloads`
**File:** `models/webhook_models/webhook_payload.go`

| Field | Type | Description |
|-------|------|-------------|
| ID | uint | PK |
| ModuleName | string | NOT NULL. Module this mapping is for |
| PayloadMapping | JSONB | NOT NULL. Maps outgoing keys to dot-notation field paths |
| IncludeEventMeta | bool | default: false. Adds `_event_meta` to payload |
| CreatedAt | time.Time | Auto |

**PayloadMapping example:**
```json
{
  "order_id": "order_id",
  "customer_email": "customer.email",
  "total_amount": "total_amt",
  "tracking_status": "tracking_status"
}
```
Key = property name in outgoing webhook payload. Value = dot-notation path into `buildingData`.

---

## WebhookCondition

**Table:** `webhook_conditions`
**File:** `models/webhook_models/webhook_condition.go`

| Field | Type | Description |
|-------|------|-------------|
| ID | uint | PK |
| WebhookID | uint | NOT NULL, FK → webhooks |
| ModuleName | string | NOT NULL (auto-filled from parent webhook) |
| FieldPath | string | NOT NULL. Dot-notation path (e.g., "customer.phone") |
| Operator | string | NOT NULL. One of: `==`, `!=`, `>`, `<`, `>=`, `<=`, `contains`, `in` |
| Value | string | NOT NULL. Comparison value |
| ConditionGroup | int | NOT NULL, default: 1. AND within group, OR across groups |
| CreatedAt | time.Time | Auto |

**Logic:** `(group1_cond1 AND group1_cond2) OR (group2_cond1) OR (group3_cond1 AND group3_cond2)`

**Operators:**
| Operator | Type | Example |
|----------|------|---------|
| `==` | String equality | `payment_mode == "COD"` |
| `!=` | String inequality | `source != "MOBILE"` |
| `>`, `<`, `>=`, `<=` | Numeric comparison | `total_amt > 1000` |
| `contains` | Substring match | `customer.email contains "@gmail"` |
| `in` | Comma-separated list | `tracking_status in "rto,rto in transit"` |

---

## WebhookLog

**Table:** `webhook_logs`
**File:** `models/webhook_models/webhook_log.go`

| Field | Type | Description |
|-------|------|-------------|
| ID | uint | PK |
| WebhookID | uint | NOT NULL, FK → webhooks |
| OrderID | string | NOT NULL. Extracted from trigger data |
| Success | bool | default: false |
| AttemptCount | int | default: 0. Total attempts made |
| MaxAttempts | int | NOT NULL. From webhook's MaxRetries |
| TotalTimeTakenMs | int | Total execution time (ms) |
| SuccessPayloadSnapshot | JSONB | Payload that succeeded (null if all failed) |
| AuthID | *uint | Optional. Auth config used for audit |
| FailedLogs | JSONB | Array of FailedLog objects |
| CreatedAt | time.Time | Auto |
| CompletedAt | *time.Time | Set on success |

**FailedLog (embedded in FailedLogs JSONB array):**

| Field | Type | Description |
|-------|------|-------------|
| AttemptNumber | int | 1-indexed attempt number |
| FailureReason | string | Error message |
| HTTPStatus | int | HTTP status code (0 if connection error) |
| TimeTakenMs | int | Time for this attempt (ms) |
| AttemptedAt | time.Time | Timestamp |
| PayloadSnapshot | JSON | Exact payload sent |

---

## WebhookEvent

**Table:** `webhook_events`
**File:** `models/webhook_models/webhook.go`

| Field | Type | Constraints |
|-------|------|-------------|
| ID | uint | PK |
| Name | string | UNIQUE, NOT NULL |

Many-to-many with Webhook via `webhook_subscribed_events` join table.

Known events: `order.rto`

---

## WebhookSource

**Table:** `webhook_sources`
**File:** `models/webhook_models/webhook.go`

| Field | Type | Constraints |
|-------|------|-------------|
| ID | uint | PK |
| Name | string | UNIQUE, NOT NULL |

Many-to-many with Webhook via `webhook_allowed_sources` join table.

Empty sources on a webhook = allow all sources.

---

## Join Tables (Auto-created by GORM)

| Table | Columns | Purpose |
|-------|---------|---------|
| `webhook_subscribed_events` | webhook_id, webhook_event_id | Webhook ↔ Event |
| `webhook_allowed_sources` | webhook_id, webhook_source_id | Webhook ↔ Source |

---

## Entity Relationship

```
WebhookAuth ←──(0..1)── Webhook ──(0..1)──→ WebhookPayload
                            │
                            ├──(1..*)──→ WebhookCondition
                            │
                            ├──(M2M)───→ WebhookEvent
                            │
                            └──(M2M)───→ WebhookSource

Webhook ──(1..*)──→ WebhookLog
                       └── FailedLog[] (embedded JSONB)
```
