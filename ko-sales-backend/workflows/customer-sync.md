---
workflow: Customer Sync
status: completed
last_worked: 2026-02-21
tags: [#completed, #webhook, #cross-project]
models: [customer]
modules: [customers]
connected_projects: [Inventory-Management-Backend]
created: 2026-02-21
---

# Customer Sync

Inventory-Management-Backend → ko-sales-backend webhook.

## Flow
```
Order delivered (status=15) in Inventory-Management-Backend
→ Aggregate by pii_id: total_order_value, max_order_value, order_count
→ POST /customers/webhook/upsert
   Header: x-webhook-secret: WEBHOOK_SECRET
   Body: full customer fields
→ Exists (by pii_id)? → update all fields
   New? → create CT-N
→ Emit entity.updated (for cohort evaluation)
```

## Auth
- @Public (no JWT)
- x-webhook-secret header validated against WEBHOOK_SECRET env
- Same pattern as /leads/upsert webhook
