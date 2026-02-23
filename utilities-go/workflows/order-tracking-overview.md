# Order Tracking System — Overview

`#active` `#cron` `#cross-project`

---

## What It Does

Cron-based system that polls courier APIs (Delhivery, Shiprocket, Velocity), maps raw courier statuses to canonical statuses, and updates order records in MongoDB. Runs in production only (`ENV=PROD`).

## Architecture

```
                   ┌─────────────────────────────────┐
                   │  main.go — Cron Scheduler        │
                   │  (robfig/cron, per-tracker mutex) │
                   └──────────┬──────────────────────┘
                              │
         ┌────────────────────┼────────────────────────┐
         │                    │                         │
    Every 8 min          Every 8 min              Every 2 hrs
         │                    │                         │
    ┌────┴────┐          ┌────┴────┐              ┌─────┴─────┐
    │B2C      │          │B2B      │              │Fallback   │
    │Shiprocket│         │Velocity │              │           │
    └────┬────┘          └────┬────┘              └─────┬─────┘
         │                    │                         │
         ▼                    ▼                         ▼
    Courier APIs         Courier APIs             Delhivery
    (batch/single)       (batch/single)           Fallback API
         │                    │                         │
         ▼                    ▼                         ▼
    Status Mapping       Status Mapping           Status Mapping
         │                    │                         │
         └────────────────────┼─────────────────────────┘
                              │
                              ▼
                   ┌──────────────────────┐
                   │  MongoDB Updates      │
                   │  - orders collection  │
                   │  - orderTracking      │
                   │  - status (read-only) │
                   └──────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         Slack Alerts    RTO Webhook      Email Report
         (lost/destroy/  (→ CRM cancel)   (SMTP summary)
          cancelled)
```

## 5 Tracking Functions

| Function | Courier | Cron | API Style | Max Orders | Workers |
|----------|---------|------|-----------|-----------|---------|
| TrackDelhiveryAndUpdateOrders | Delhivery B2C | */8 * * * * | Batch GET, 50 AWBs | 36,000 | 50 |
| TrackDelhiveryB2BAndUpdateOrders | Delhivery B2B | */8 * * * * | Single GET per LR | 480 | 3 |
| TrackShiprocketAndUpdateOrders | Shiprocket | */8 * * * * | Batch POST, 50 AWBs | All | 50 |
| TrackVelocityAndUpdateOrders | Velocity | */8 * * * * | Batch POST, 45 AWBs | 10,000 | Batch |
| TrackDelhiveryFallbackOrders | Delhivery Fallback | 0 */2 * * * | Single GET, 5s delay | All | 3 |

All 5 also run immediately at server startup. Each guarded by `sync.Mutex` + `TryLock()` — cron skips if previous run still active.

## Canonical Statuses

```
pending → in transit → out_for_delivery → delivered
                    ↘ ndr
                    ↘ rto → rto in transit
                    ↘ lost
                    ↘ destroy
                    ↘ booking_cancelled
```

## Key Files

| File | Purpose | ~Lines |
|------|---------|--------|
| services/order_tracking.go | All 5 tracking functions | ~3000 |
| utils/tracking_helper.go | MongoDB filters, UpdateOrderTrackingWithHistory | ~250 |
| utils/delhivery_status_mapping.go | B2C StatusType + legacy mapping | ~200 |
| utils/shiprocket_status_mapping.go | Shiprocket status mapping | ~100 |
| utils/velocity_status_mapping.go | Velocity status mapping | ~80 |
| utils/delhivery_response_helper.go | B2B API call + response parsing | ~150 |
| services/send_slack.go | Slack notifications | ~100 |
| services/tracking_logger.go | Buffered alert logging → S3 | ~200 |
| services/tracking_query_logger.go | Query logging → S3 + Slack | ~100 |
| services/rto_webhook.go | RTO → CRM webhook trigger | 97 |

## Related Docs

- [Courier API Details](order-tracking-courier-apis.md)
- [Status Mapping Reference](order-tracking-status-mapping.md)
- [RTO Webhook Integration](rto-webhook-integration.md)
