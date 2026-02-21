---
workflow: SLA Engine
status: completed
last_worked: 2026-02-21
tags: [#completed, #active, #cron]
models: [task]
modules: [tasks]
---

# SLA Engine

Business hours calculation for task deadlines.

## Config
- Hours: 10:00 AM – 7:00 PM (9h/day)
- Days: Mon – Sat (Sunday off)
- At-risk threshold: 20% remaining

## Parsing
- `"2h"` → 120 business-minutes
- `"1d"` → 540 business-minutes
- `"48h"` → 2880 business-minutes

## Deadline Example
```
Created: 5PM Mon, SLA: 4h
→ 2h Mon (5-7PM) + 2h Tue (10AM-12PM)
→ Deadline: Tue 12:00 PM
```

## Status Transitions
```
on_track → at_risk (20% remaining) → breached (past deadline)
any → completed (task done, records breach_duration if breached)
```

## Endpoints
- `GET /tasks/:id/sla` — single task SLA check
- `POST /tasks/sla/check-all` — batch update all active tasks

## Cron
- FollowUpCronService: every 30 min, marks overdue follow_up tasks

## Utils
`src/common/utils/sla.util.ts` — pure functions: parse_sla, is_business_hours, calculate_sla_deadline, calculate_breach_duration, count_business_minutes
