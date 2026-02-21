---
workflow: Assignment Cascade
status: completed
last_worked: 2026-02-21
tags: [#completed, #active, #core]
models: [lead, contact, pipeline, pipeline-workflow, agent]
modules: [leads, contacts, pipelines, agents]
---

# Assignment Cascade

`POST /leads/:id/assign` → `orchestrate_assign(lead_id, agent_id, assigned_by)`

## Flow

```
Validate agent → get agent_category
        │
   Contact exists for PII?
   ┌─────┴─────┐
 [YES]        [NO]
   │            │
Close old    Close old
CP-* pipe    LP-* pipe
   │            │
Update       Create new
contact      LP-* pipe
agent_cat    (new agent)
   │            │
Create new   Set on
CP-* pipe    lead.pipeline_id
   │            │
Set on       ───┘
contact.     │
pipeline_id  │
   │         │
   └────┬────┘
        │
Update lead:
  lead_owner → {agent_id, agent_ref}
  assigned_date → now
  assigned_by → user
  audit trail entry
        │
Deep populate response:
  lead + PII + agent + pipeline + workflow
```

## Key Rules

- Contact pipeline takes priority if contact exists
- Old pipeline CLOSED (not deleted), reason: `owner_changed`
- New pipeline gets agent's workflow config
- No bulk assign yet — `#needs-discussion` for cohort assignment
