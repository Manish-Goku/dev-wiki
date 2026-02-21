---
workflow: Lead Lifecycle
status: completed
last_worked: 2026-02-21
tags: [#completed, #active, #core]
models: [lead, pii, contact, pipeline, agent, note, signal]
modules: [leads, pii, contacts, pipelines, agents, notes, signals]
---

# Lead Lifecycle

## Create Flow
```
Phone arrives → PII check (exists? reuse : create)
→ Duplicate lead check (409 if exists)
→ Owner resolution: explicit > round-robin (state+category) > default > null
→ Create lead (K0-1XXXXX, status=active)
→ Pipeline creation (if owner + agent_category → LP-N)
→ Contact creation (if profile provided → C-N + CP-N, status=contact_created)
→ Notes creation (if notes[] provided)
```

## Update Flow
```
Fields separated into 3 buckets:
├── PII: phone, email, address, gst → PiiService.update
├── Contact: profile, profile_details → ContactsService.update/create
└── Lead: everything else → apply_update

InstructionProcessor: tracks tags, source, disposition, lead_owner → audit trail
Call status change → increment call_count + lead_activity entry
Owner change → pipeline cascade (close old, create new)
Contact exists? → update contact fields
No contact + profile? → create contact + CP-* pipeline
```

## Upsert Flow (Webhook)
```
POST /leads/upsert (@Public, x-webhook-secret)
Phone lookup → exists? update fields + create duplicate signal
           → new? create PII + lead + pipeline + note from message
```
