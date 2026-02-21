# Inventory-Management-Backend

## Overview
Backend service for inventory and order management at Katyayani Organics. Handles order lifecycle, inventory tracking, shipping integrations, and promotional freebies.

## Tech Stack
- **Runtime:** Node.js, Express.js
- **Database:** MongoDB (Mongoose ODM), shared `CRM-Database`
- **Queue:** BullMQ (Redis-backed background jobs)
- **Realtime:** Socket.io with Redis adapter
- **Clustering:** `Cluster.js` for multi-core utilization

## Entry Points
- `Cluster.js` → forks `app.js` per CPU core
- `start-worker.js` → BullMQ worker processes

## Route Loading
Auto-discovers `route.js` files in `routes/*/` subdirectories via `routes/index.js`.
All routes mounted at `/api/<directory-name>/*`.

## Modules

| Module | Path | Docs |
|--------|------|------|
| [Freebies](workflows/freebies.md) | `routes/freebies/` | Rule-based promotional freebie system |

## Models

| Model | Collection | Doc |
|-------|-----------|-----|
| [FreebieRule](models/freebie-rule.md) | `freebie_rules` | Eligibility rules with json-rules-engine conditions |
| [FreebieUsageLog](models/freebie-usage-log.md) | `freebie_usage_logs` | Distribution tracking per customer/order |
| [AgentFreebieQuota](models/agent-freebie-quota.md) | `agent_freebie_quotas` | Per-agent distribution limits |
| [PromotionalItem](models/promotional-item.md) | `promotional_items` | Freebie item definitions (product/combo) |
