---
workflow: Task Automation
status: completed
last_worked: 2026-02-21
tags: [#completed, #active, #sla, #supabase]
models: [task, task-workflow, pipeline, note]
modules: [tasks, pipelines, notes]
---

# Task Automation

Pipeline events → auto-create tasks → SLA tracking → auto-advance pipeline.

## Flow

```
Pipeline created / Stage transitioned
→ Cancel old stage pending tasks
→ Stage has taskflow_id?
  NO → skip
  YES → Duplicate check (step 1 exists for pipeline+workflow?)
    YES → skip
    NO → Create task (T-N):
         • Calculate SLA deadline (business hours)
         • Set sla_started_at, sla_status=on_track
         • Sync to Supabase (optional)

Agent completes task (POST /:id/complete)
→ Mark completed + SLA complete
→ Optional: create linked Note (bidirectional)
→ Next step exists?
  YES → Create next step task (with dedup)
  NO → All steps done → auto_advance_pipeline()
       → Force transition to next stage
       → Trigger new stage tasks
```

## SLA Engine

- Business hours: 10:00 AM – 7:00 PM Mon–Sat (9h/day)
- Parse: "2h" → 120 min, "1d" → 540 min
- Statuses: on_track → at_risk (20% remaining) → breached → completed
- Endpoints: `GET /tasks/:id/sla`, `POST /tasks/sla/check-all`
- Cron: FollowUpCronService every 30 min marks overdue follow_up tasks
