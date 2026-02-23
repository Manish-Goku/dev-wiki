# ivr_calls

IVR call records — covers live calls, hangups, missed, completed. Single table powering all IVR pages.

## Key Columns

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Auto-generated |
| `call_id` | text | Display call ID (e.g., "IVR-GEN-001") |
| `mobile_number` | text | Customer phone |
| `customer_name` | text | Customer name (nullable) |
| `state` | text | Customer state/location |
| `department` | text | `general`, `retailer`, `app`, `kyc`, `international` |
| `status` | text | `waiting`, `ringing`, `active`, `missed`, `hangup`, `completed` |
| `agent_id` | uuid (FK → agents) | Assigned agent |
| `agent_name` | text | Denormalized agent name |
| `did_number` | text | DID number called |
| `priority` | text | `normal`, `high`, `vip`, `repeat` |
| `received_at` | timestamptz | When call came in |
| `answered_at` | timestamptz | When agent answered |
| `ended_at` | timestamptz | When call ended |
| `duration_seconds` | int | Call duration |
| `hold_time_seconds` | int | Hold time |
| `category` | text | Disposition category |
| `sub_category` | text | Disposition sub-category |
| `resolution` | text | `solved`, `escalated`, `callback`, `transfer` |
| `customer_sentiment` | text | `happy`, `neutral`, `frustrated` |
| `remark` | text | Agent notes |
| `recording_url` | text | Call recording URL |
| `order_id` | text | Related order (nullable) |
| `attempts` | int | Callback attempt count (hangup calls) |
| `last_attempt_at` | timestamptz | Last callback attempt |
| `is_sla_breached` | boolean | SLA breach flag |
| `sla_breach_at` | timestamptz | When SLA was breached |
| `action` | text | e.g., "Solved" |
| `customer_tier` | text | Customer tier |
| `entry_date` | date | Entry date |

## Status Flow

```
waiting → ringing → active → completed
waiting → missed (unanswered)
waiting → hangup (customer hung up before answer)
hangup → assigned → in_progress → completed (callback flow)
hangup → not_responded (3 failed callback attempts)
```

## Queried By

| Hook/Page | Filter |
|-----------|--------|
| `useIVRCalls` (CallHistory) | `status IN ('completed','missed','hangup')` |
| `useIVRCalls` (IVRConsolidated) | No filter (all statuses) |
| `useIVRCalls` (SLABreach) | `is_sla_breached = true` |
| `useIVRCalls` (IVRLiveDepartment) | `status IN ('waiting','ringing','active','missed') AND department = ?` |
| `useIVRCalls` (IVRHangup) | `status = 'hangup'` |
| `useIVRSidebarCounts` | All rows — groups client-side by status + department |
| `useAgentCalls` | `status IN ('active','ringing') AND agent_id = ?` |
| `useAgentHangups` | `status = 'hangup' AND agent_id = ?` |

## Notes

- `agent_id` is UUID type — must match `agents.id`, not string IDs like 'ag-1'
- Real-time subscriptions via `postgres_changes` on this table power all IVR pages
- Sidebar counts: live = `waiting + ringing + missed`, hangup = `hangup` status per department
