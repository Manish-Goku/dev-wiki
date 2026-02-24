# kaleyra_call_logs

Synced call data from Kaleyra Voice API. Each row is one call leg (Incoming to DID, CallForward to agent, or Click2Call).

## Schema

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | TEXT | NO | — | **PK.** Kaleyra call ID (UUID format) |
| `uniqid` | TEXT | YES | — | Kaleyra internal unique ID (e.g. `58_1771925707.21729277`) |
| `callfrom` | TEXT | NO | — | Caller phone number |
| `callto` | TEXT | NO | — | DID number (8068921234) or agent number |
| `start_time` | TIMESTAMPTZ | NO | — | Call start timestamp |
| `end_time` | TIMESTAMPTZ | YES | — | Call end timestamp |
| `duration` | INTEGER | NO | 0 | Total duration in seconds |
| `billsec` | INTEGER | NO | 0 | Billable/talk duration in seconds |
| `status` | TEXT | NO | — | ANSWER, NOANSWER, BUSY, CANCEL, FAILED, CONGESTION, ANSWERED-A, ANSWERED-B, ACCEPTED, FORWARDED |
| `location` | TEXT | YES | — | Caller state/circle (BIHAR, UP EAST, MADHYA PRADESH, WEST BENGAL, etc.) |
| `provider` | TEXT | YES | — | Telecom carrier (JIO, AIRTEL, IDEA, Tata Teleservices, etc.) |
| `service` | TEXT | NO | — | Call type: `Incoming`, `CallForward`, `Click2Call` |
| `caller_id` | TEXT | YES | — | DID used for the call |
| `recording` | TEXT | YES | — | Recording URL (e.g. `//play.solutionsinfini.com/?id=...`) |
| `notes` | TEXT | YES | — | Call notes |
| `custom` | TEXT | YES | — | Custom data field |
| `synced_at` | TIMESTAMPTZ | NO | now() | When this record was synced from Kaleyra |

## Indexes

| Index | Column(s) | Purpose |
|-------|-----------|---------|
| `idx_kaleyra_logs_start_time` | start_time | Date range queries |
| `idx_kaleyra_logs_service` | service | Filter by Incoming/CallForward/Click2Call |
| `idx_kaleyra_logs_callto` | callto | Agent number lookups |
| `idx_kaleyra_logs_status` | status | Status filtering |

## RLS

Row-level security enabled. Single policy: `Allow all for service role` — backend uses service_role key.

## RPC Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `get_kaleyra_call_volume(p_start, p_end)` | `{total, incoming, forwarded, click2call}` | Aggregated call counts by service type |
| `get_kaleyra_agent_stats(p_start, p_end)` | `[{agent_number, calls_answered, total_talk_seconds, missed_calls}]` | Per-agent performance (CallForward legs only) |
| `get_kaleyra_daily_volume(p_start, p_end)` | `[{date, incoming, forwarded}]` | Daily volume trend (Asia/Kolkata timezone) |
| `get_kaleyra_status_breakdown(p_start, p_end)` | `[{status, count}]` | Status distribution |

## Call Flow

```
Customer dials 8068921234 (IVR DID)
  → Incoming leg: callfrom=customer, callto=8068921234
    → IVR greeting plays
    → CallForward leg 1: callfrom=customer, callto=9201952685 (agent 1)
      → If no answer → CallForward leg 2: callfrom=customer, callto=9201972062 (agent 2)
        → If no answer → CallForward leg 3: callfrom=customer, callto=6264437618 (agent 3)
```

One customer call generates **1 Incoming + N CallForward** rows (N = number of agents tried).

## Data Source

Synced via `POST /calls/sync` → Kaleyra API `method=dial` with date range pagination → upsert on `id` conflict.

## Related

- [ivr_calls](ivr_calls.md) — webhook-ingested IVR data (different from synced Kaleyra logs)
- [ivr_providers](ivr_providers.md) — dynamic IVR provider registry
