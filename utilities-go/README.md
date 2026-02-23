# utilities-go — Dev Wiki

Golang backend service for Katyayani Organics. Handles order tracking crons, configurable webhooks, and utility APIs.

**Stack:** Go, Gin, MongoDB (orders/marketplaces) + PostgreSQL (webhooks via GORM)
**Port:** 5000
**Entry:** `main.go`

## Modules

### Webhook System
Configurable event-driven webhook engine stored in PostgreSQL. Supports conditional execution, dynamic payload mapping, dual authentication (static headers / OAuth2 token), JSON schema validation, and manual retry.

- [Webhook Models](models/webhook-models.md) — All PostgreSQL schemas (Webhook, Auth, Payload, Condition, Log, Event, Source)
- [Webhook Execution Flow](workflows/webhook-trigger-flow.md) — End-to-end trigger pipeline
- [Webhook API Reference](workflows/webhook-api-reference.md) — All REST endpoints
- [RTO Webhook Integration](workflows/rto-webhook-integration.md) — How tracking crons trigger webhooks

### Order Tracking
Cron-based system polling 4 courier APIs (Delhivery B2C/B2B, Shiprocket, Velocity) to update order statuses in MongoDB. 5 tracking functions, production only.

- [Order Tracking Overview](workflows/order-tracking-overview.md) — Architecture, all 5 functions, key files
- [Courier API Details](workflows/order-tracking-courier-apis.md) — Endpoints, auth, batch sizes, response formats, rate limiting
- [Status Mapping Reference](workflows/order-tracking-status-mapping.md) — All courier-to-canonical mappings, tracking_timestamps
- [MongoDB Collections](models/order-tracking-collections.md) — orders, orderTracking, status collections, update patterns
- [RTO Webhook Integration](workflows/rto-webhook-integration.md) — RTO status → CRM cancel-order webhook

### Pincode Case Pricing
Zone-based pricing system for product cases. Different geographical zones get different pricing tiers, identified by pincodes. Managed here via REST API, consumed by Inventory-Management-Backend's search product API via shared MongoDB.

- [Pincode Case Models](models/pincode-case-models.md) — MongoDB schemas (pricing, audit log), indexes, entity relationships
- [Pincode Case Pricing Workflow](workflows/pincode-case-pricing.md) — Full API reference, search product integration, UOM conversion, business rules

### Cron Jobs (Dynamic HTTP Scheduler)
REST API for managing scheduled HTTP jobs. Jobs are persisted in PostgreSQL and executed on cron schedules, firing authenticated HTTP requests to configured URLs. SSO Bearer token auto-managed.

- [Cron Job Models](models/cron-job-models.md) — PostgreSQL schema, in-memory state, cron spec generation
- [Cron Jobs Workflow](workflows/cron-jobs.md) — API reference, execution flow, SSO token manager, lifecycle

### Token Service
Centralized API for backend services to obtain third-party auth tokens. Dynamic tokens (Shiprocket, Delhivery B2B) with thread-safe caching and refresh. Static tokens (Delhivery B2C, Razorpay) with IP whitelist.

- [Token Service](workflows/token-service.md) — API reference, platform details, token managers, IP whitelist, env vars

## Key Files

| File | Purpose |
|------|---------|
| `main.go` | Entry point, cron setup, DB init |
| `routes/route.go` | Manual route registration (`RegisterRoutes()`) |
| `database/postgress.go` | PostgreSQL + GORM init, AutoMigrate |
| `database/mongo.go` | MongoDB connection |
| `services/order_tracking.go` | All tracking cron functions |
| `services/webhook_trigger.go` | Core webhook execution engine |
| `services/rto_webhook.go` | RTO-specific webhook trigger |

## Tags
`#active` `#cron` `#webhook` `#auth-required`
