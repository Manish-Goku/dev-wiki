# IVR Supabase Wiring

How the frontend IVR pages connect to the `ivr_calls` Supabase table.

## Architecture

```
Supabase (ivr_calls table)
    ↕ postgres_changes (real-time)
    ↕ REST API (CRUD)
    │
    ├─ useIVRCalls(params)          ← Unified hook (all 7 pages)
    │   ├─ CallHistory              ← statuses: completed, missed, hangup
    │   ├─ IVRConsolidated          ← all statuses (admin view)
    │   ├─ SLABreach                ← slaOnly: true
    │   ├─ IVRLiveDepartment        ← statuses: waiting, ringing, active, missed + department
    │   ├─ IVRLive                  ← statuses: waiting, ringing, active
    │   ├─ IVRHangup                ← statuses: hangup
    │   └─ IVRHangupDepartment      ← statuses: hangup + department
    │
    └─ useIVRSidebarCounts()        ← Sidebar badges (lightweight, separate query)
        ├─ IVR Live per dept        ← waiting + ringing + missed
        ├─ IVR Hangup per dept      ← hangup status
        ├─ SLA Breach total         ← is_sla_breached = true
        └─ All Total                ← total row count
```

## useIVRCalls — Unified Hook

### Params
```typescript
{
  statuses?: string[];      // Filter by status IN (...)
  department?: string;      // Filter by department = ...
  slaOnly?: boolean;        // Filter by is_sla_breached = true
  agentId?: string;         // Filter by agent_id = ...
  limit?: number;           // Default 500
  orderBy?: { column: string; ascending?: boolean };
}
```

### Returns
```typescript
{
  calls: IVRCallRow[];      // Mapped from DB snake_case → camelCase
  loading: boolean;
  stats: IVRCallStats;      // Computed client-side: total, waiting, active, completed, missed, hangup, slaBreached, avgDuration
  updateCall(id, updates);  // Generic update
  assignCall(id, agentId, agentName);  // Assign to agent
  bulkAssign(ids, agentId, agentName); // Bulk assign
  logAttempt(id);           // Increment attempts, auto-mark not_responded at 3
  endCall(id, disposition); // Complete with disposition data
  refresh();                // Manual re-fetch
}
```

### Key Mapping
| DB Column | Frontend Field |
|-----------|---------------|
| `mobile_number` | `phone` |
| `duration_seconds` | `duration` |
| `hold_time_seconds` | `holdTime` |
| `customer_sentiment` | `sentiment` |
| `sub_category` | `subCategory` |
| `remark` | `notes` |
| `is_sla_breached` | `isSlaBreached` |

## Adapter Functions

### toHangupCall() — IVRHangup.tsx
Converts `IVRCallRow` → `HangupCall` (component type):
- Derives status: `action='Solved'` → closed, `attempts>=3` → not_responded, `isSlaBreached` → sla_breach, `attempts>0` → in_progress, `agentId` → assigned, else → pending
- Maps priority: vip/high → 'high', repeat → 'repeat_caller'
- Builds synthetic attempts array from `attempts` count

### callToBreach() — SLABreach.tsx
Converts `IVRCallRow` → breach display format:
- `deriveSeverity()`: priority + breach duration → critical/high/medium/low
- `deriveBreachStatus()`: call status → active/resolved/escalated

## Sidebar Count Logic

**Important**: Sidebar live count uses `['waiting', 'ringing', 'missed']` — NOT `active`.
- `active` calls are already being handled by an agent
- `missed` calls need attention (callback required)
- This ensures sidebar badge matches what the department page shows

## Gotcha: agent_id is UUID

The `agent_id` column is `uuid` type referencing `agents.id`. When seeding test data or assigning agents, use actual UUIDs from the `agents` table, not string IDs like 'ag-1'.
