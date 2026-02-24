# Kaleyra Call Dashboard

End-to-end flow for syncing Kaleyra Voice call data and displaying dashboard analytics.

## Overview

Kaleyra handles all IVR calls for Katyayani Organics. The dashboard syncs call logs from Kaleyra API into Supabase, then serves aggregated metrics via dashboard endpoints.

## Sync Flow

```
POST /calls/sync { from_date?: "YYYY/MM/DD", to_date?: "YYYY/MM/DD" }
  │
  ├── Default: today's date for both
  │
  ├── Loop (paginated, 200 records/page):
  │   POST https://api-voice.kaleyra.com/v1/
  │     method=dial
  │     api_key=<KALEYRA_API_KEY>
  │     fromdate=<from>
  │     todate=<to>
  │     page=<N>
  │     limit=200
  │     format=json
  │
  │   → Parse response.data[] or response.records[]
  │   → Map fields: callfrom, callto, start_time, end_time, duration, billsec,
  │     status, location, provider, service, recording
  │   → Parse datetime: "2026-02-24 14:45:15" → ISO with +05:30 offset
  │   → Upsert into kaleyra_call_logs (ON CONFLICT id)
  │   → Continue until page has < 200 records
  │
  └── Return { synced: <total_count>, status: "ok" }
```

## Dashboard Flow

```
GET /calls/dashboard/overview?range=today|7d|30d|custom&start_date=&end_date=
  │
  ├── resolve_dates(dto) → { start_date, end_date } (ISO timestamps)
  │   today  → midnight local → now
  │   7d     → now - 7 days → now
  │   30d    → now - 30 days → now
  │   custom → dto.start_date → dto.end_date
  │
  └── Promise.all([
        rpc('get_kaleyra_call_volume', p_start, p_end),      → { total, incoming, forwarded, click2call }
        rpc('get_kaleyra_agent_stats', p_start, p_end),      → [{ agent_number, calls_answered, total_talk_seconds, missed_calls }]
        rpc('get_kaleyra_daily_volume', p_start, p_end),     → [{ date, incoming, forwarded }]
        rpc('get_kaleyra_status_breakdown', p_start, p_end), → [{ status, count }]
        get_last_synced_at(),                                  → most recent synced_at from table
      ])
      → Return combined CallDashboardOverviewDto
```

## Endpoints

| Method | Path | Query Params | Response |
|--------|------|--------------|----------|
| POST | `/calls/sync` | Body: `{from_date?, to_date?}` | `{synced, status}` |
| GET | `/calls/dashboard/overview` | range, start_date, end_date | `CallDashboardOverviewDto` |
| GET | `/calls/dashboard/agent-stats` | range, start_date, end_date | `AgentStatsDto[]` |
| GET | `/calls/dashboard/daily-volume` | range, start_date, end_date | `DailyCallVolumeDto[]` |
| GET | `/calls/dashboard/call-list` | range, start_date, end_date, service, agent_number, page, limit | `{data, total, page}` |

## Agent Stats Logic

Only counts `service = 'CallForward'` rows (agent legs):
- **calls_answered**: COUNT where status = 'ANSWER'
- **total_talk_seconds**: SUM of billsec where status = 'ANSWER'
- **missed_calls**: COUNT where status IN ('NOANSWER', 'BUSY', 'CANCEL', 'FAILED')

## Known Agent Numbers

| Number | Role |
|--------|------|
| 8068921234 | IVR DID (main number customers dial) |
| 9201952685 | Agent — first in forward chain |
| 9201972062 | Agent |
| 6264437618 | Agent |
| 9201836530 | Agent |
| 9238109518 | Agent |
| 9201972072 | Agent — appears inactive (0 answered in 7d) |

## Status Values

| Status | Meaning | Count (7d sample) |
|--------|---------|-------------------|
| ANSWER | Call answered | 1233 |
| NOANSWER | Ring timeout, no pickup | 313 |
| ANSWERED-B | Answered on B-leg (forwarded) | 190 |
| FAILED | Call failed to connect | 138 |
| ANSWERED-A | Answered on A-leg | 99 |
| CANCEL | Caller hung up before answer | 88 |
| ACCEPTED | Call accepted (IVR level) | 16 |
| FORWARDED | Call forwarded to next agent | 7 |
| BUSY | Agent busy | 2 |
| CONGESTION | Network congestion | 2 |

## Files

| File | Purpose |
|------|---------|
| `src/kaleyraVoice/kaleyraVoice.service.ts` | sync_call_logs(), dashboard methods, RPC wrappers |
| `src/kaleyraVoice/kaleyraVoice.controller.ts` | 5 dashboard routes |
| `src/kaleyraVoice/dto/callDashboard.dto.ts` | Query + response DTOs |

## Related

- [kaleyra_call_logs model](../models/kaleyra_call_logs.md) — table schema
- Existing endpoints: `/calls/click-to-call`, `/calls/outbound`, `/webhooks/kaleyra/callback`, `/calls/logs`
