# Webhook API Reference

All endpoints require JWKS authentication.

`#active` `#webhook` `#auth-required`

---

## Webhooks — `/webhooks`

### Create Webhook
`POST /webhooks`

```json
{
  "url": "https://api.example.com/hook",
  "module_name": "order",
  "auth_id": 5,
  "payload_id": 3,
  "registered_by": "admin@company.com",
  "description": "Notify on RTO orders",
  "is_active": true,
  "retry_enabled": true,
  "max_retries": 3,
  "conditions": [
    {
      "field_path": "tracking_status",
      "operator": "in",
      "value": "rto,rto in transit",
      "condition_group": 1
    }
  ]
}
```

**Validation:**
- `conditions` array is mandatory (>= 1 required)
- Each condition: `field_path`, `operator`, `value` must be non-empty
- `operator` must be one of: `==`, `!=`, `>`, `<`, `>=`, `<=`, `contains`, `in`
- `module_name` auto-filled on conditions from webhook's module_name

**Response:** 201 with full webhook object

### List Webhooks
`GET /webhooks?module_name=order&is_active=true`

Optional query filters: `module_name`, `is_active` ("true"/"false")

### Get Webhook
`GET /webhooks/:id`

Returns webhook with preloaded Events, Sources, Conditions.

### Update Webhook
`PUT /webhooks/:id`

### Delete Webhook
`DELETE /webhooks/:id`

---

## Conditions — `/webhooks/:id/conditions`

### Create Condition
`POST /webhooks/:id/conditions`

```json
{
  "field_path": "payment_mode",
  "operator": "==",
  "value": "COD",
  "condition_group": 2
}
```

### List Conditions
`GET /webhooks/:id/conditions`

Ordered by `condition_group`.

### Update Condition
`PUT /webhooks/:id/conditions/:conditionId`

### Delete Condition
`DELETE /webhooks/:id/conditions/:conditionId`

---

## Logs — `/webhooks/:id/logs`

### List Webhook Logs
`GET /webhooks/:id/logs`

Returns all execution logs for a webhook, ordered by `created_at DESC`.

### Retry Failed Webhook
`POST /webhooks/logs/:id/retry`

Optional request body:
```json
{
  "payload": {
    "order_id": "ORD-2026-001",
    "customer_email": "john@example.com"
  }
}
```

If no payload provided, uses the last failed attempt's payload snapshot.

---

## Manual Trigger — `/webhooks/trigger`

### Trigger Webhooks
`POST /webhooks/trigger`

```json
{
  "module_name": "order",
  "source": "MOBILE",
  "event": "order.rto",
  "updated_data": {
    "tracking_status": "rto",
    "order_id": "ORD-001"
  },
  "building_data": {
    "order_id": "ORD-001",
    "customer": {"email": "john@example.com", "phone": "9876543210"},
    "total_amt": 1500,
    "source": "Shopify",
    "status": "rto",
    "tracking_status": "rto"
  }
}
```

**Response:**
```json
{
  "triggered_webhooks": 2,
  "failed_webhooks": 0,
  "results": [
    {"webhook_id": 5, "webhook_url": "https://...", "success": true, "time_taken_ms": 250, "log_id": 42}
  ]
}
```

---

## Auth — `/webhook-meta/auth`

### Create Auth
`POST /webhook-meta/auth`

**Static headers mode:**
```json
{
  "headers": {"Authorization": "Bearer sk_live_abc123", "X-API-Key": "key123"}
}
```

**Token-based (OAuth2) mode:**
```json
{
  "auth_url": "https://auth.example.com/oauth/token",
  "auth_payload": {"client_id": "id", "client_secret": "secret", "grant_type": "client_credentials"},
  "response_token_path": "access_token"
}
```

### List Auths
`GET /webhook-meta/auth?page=1&limit=20`

Paginated. Response: `{ data: [...], pagination: { page, limit, total, total_pages } }`

### Get Auth
`GET /webhook-meta/auth/:id`

### Update Auth
`PUT /webhook-meta/auth/:id`

### Delete Auth
`DELETE /webhook-meta/auth/:id`

Blocked if referenced by any webhook.

---

## Payload — `/webhook-meta/payload`

### Create Payload
`POST /webhook-meta/payload`

```json
{
  "module_name": "order",
  "payload_mapping": {
    "phone_number": "customer.phone",
    "order_id": "order_id",
    "order_source": "source",
    "order_value": "total_amt",
    "status": "status"
  },
  "include_event_meta": false
}
```

Validates mapping against module's JSON schema.

### List Payloads
`GET /webhook-meta/payload?page=1&limit=20&module_name=order`

Paginated, filterable by `module_name`.

### Get Payload
`GET /webhook-meta/payload/:id`

### Update Payload
`PUT /webhook-meta/payload/:id`

Re-validates mapping on update.

### Delete Payload
`DELETE /webhook-meta/payload/:id`

Blocked if referenced by any webhook.

---

## Events — `/webhook-meta/events`

### Create Event
`POST /webhook-meta/events`
```json
{"name": "order.rto"}
```

### List Events
`GET /webhook-meta/events`

### Delete Event
`DELETE /webhook-meta/events/:id`

Blocked if subscribed by any webhook.

---

## Sources — `/webhook-meta/sources`

### Create Source
`POST /webhook-meta/sources`
```json
{"name": "MOBILE"}
```

### List Sources
`GET /webhook-meta/sources`

### Delete Source
`DELETE /webhook-meta/sources/:id`

Blocked if referenced by any webhook.
