---
workflow: Deal & Quotation
status: completed
last_worked: 2026-02-21
tags: [#completed, #active, #pipeline]
models: [deal, contact, pipeline]
modules: [deals, contacts, pipelines]
---

# Deal & Quotation Lifecycle

## Deal Creation
```
POST /deals → DealWorkflowService.orchestrate_create()
→ Create deal (D-N, status=open)
→ Auto-close contact pipeline (reason: deal_created)
→ Create deal pipeline (DP-N)
→ Trigger stage tasks
```

## Quotation Lifecycle
```
POST /deals/:id/quotation → add_quotation()
  → Auto-expire previous draft/shared quotations
  → Create new quotation (Q-N, status=draft)

POST /deals/:id/quotation/:qid/share → share_quotation()
  → Set status=shared
  → Move pipeline to quotation_sent stage

POST /deals/:id/negotiate → negotiate_deal()
  → Reject ALL existing quotations
  → Create new quotation (prefilled from previous)
  → Move pipeline to negotiation stage
```

## Status Flow
```
draft → shared → accepted / rejected / expired
(new quotation auto-expires old draft/shared)
```
