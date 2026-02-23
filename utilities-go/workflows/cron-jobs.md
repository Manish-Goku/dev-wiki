# Cron Jobs — Dynamic HTTP Scheduler

`#active` `#cron`

---

## Purpose

REST API for managing scheduled HTTP jobs. Users create jobs via API specifying a URL, HTTP method, start time, and repeat interval. The system generates a cron schedule, persists it in PostgreSQL, and executes HTTP requests on schedule with SSO authentication.

---

## API Reference

**Base path:** `/cron/jobs` | **Auth:** None (no JWKS middleware)

### Create Job
`POST /cron/jobs`

```json
{
  "url": "https://api.example.com/process-orders",
  "method": "GET",
  "repeatTime": "2h",
  "startTime": "09:00"
}
```

**Validation:**
- All fields required
- `startTime` must be HH:MM format
- `repeatTime` must be valid Go duration (e.g., "30m", "1h", "24h")
- 409 Conflict if job with same URL + same generated cron spec already exists

**Response:** 201 with created job including generated `cron_spec`

### List Jobs
`GET /cron/jobs`

Returns all jobs from PostgreSQL.

### Update Job
`PUT /cron/jobs/:id`

Same body as create. If cron spec changes, old cron entry is removed (if no other jobs share it) and new one registered.

### Delete Job
`DELETE /cron/jobs/:id`

Deletes from DB, removes from in-memory map, removes cron entry if no other jobs share the spec.

---

## Execution Flow

```
Cron tick fires for a spec
    │
    ├── GetJobsByCronSpec(spec) → find all jobs with that cron spec
    │
    ├── SSOToken.GetToken(1h) → get cached SSO Bearer token
    │   ├── Token valid? → return cached
    │   └── Expired/expiring? → performFullLogin() → cache new token
    │
    └── For each job:
        ├── Create HTTP request (job.Method, job.URL)
        ├── Set Authorization: Bearer {token}
        ├── Set Content-Type: application/json
        ├── Execute with 60s timeout
        └── Log result
```

---

## SSO Token Manager

**File:** `utils/sso_token_manager.go`

Thread-safe singleton that authenticates cron HTTP requests against the SSO service.

### Two-Step Login
1. **POST** `{SSO_BASE_URL}/login` → `{email, password}` → receives `code`
2. **POST** `{SSO_BASE_URL}/generate-tokens` → `{code, platform: "utility-golang", access_payload, refresh_payload}` → receives `access_token`

### Caching & Thread Safety
- Token cached in memory with JWT expiry parsed from `exp` claim
- Refreshes when expiring within 1 hour (configurable buffer)
- If multiple goroutines need token simultaneously, only one performs login (others wait via `sync.Cond`)
- Retry: 3 attempts with exponential backoff (2s, 4s, 6s)

### Methods
| Method | Purpose |
|--------|---------|
| `GetToken(buffer)` | Return cached token or refresh if expiring within buffer |
| `ForceRefresh()` | Clear cache, force new login |
| `ClearToken()` | Clear cache |
| `GetTokenInfo()` | Debug info (expiry, validity, last refresh) |

### Environment Variables
| Var | Purpose |
|-----|---------|
| `SSO_BASE_URL` | SSO service URL |
| `SSO_EMAIL` | Service account email |
| `SSO_PASSWORD` | Service account password |

---

## Startup Sequence

```go
// main.go
utils.InitCron()      // Create + start robfig/cron scheduler
utils.RestoreJobs()   // Load all jobs from PostgreSQL → populate JobsMap → register cron entries
```

Runs unconditionally (not gated by ENV=PROD like tracking crons).

---

## Cron Spec Deduplication

Multiple jobs can share the same generated cron spec. The system maintains a `CronEntriesMap[cronSpec] → entryID`:

- **Add job:** If spec already in map → skip cron registration (existing entry handles it)
- **Delete/Update job:** Check if any other jobs still use the old spec → if not, remove the cron entry

When the cron fires, `GetJobsByCronSpec()` returns ALL jobs for that spec, and they're all executed in sequence.

---

## Job Lifecycle Diagram

```
POST /cron/jobs
    │
    ├── Validate payload
    ├── GenerateCronSpec(startTime, repeatTime)
    ├── Check duplicate (URL + cronSpec)
    ├── INSERT into job_models (PostgreSQL)
    ├── Add to JobsMap (in-memory)
    └── AddJobToCron(cronSpec)
        └── If spec not in CronEntriesMap → register new cron entry

Cron tick fires
    │
    ├── Get SSO token (cached or fresh)
    └── For each job with matching spec:
        └── HTTP {method} {url} with Bearer token

DELETE /cron/jobs/:id
    │
    ├── DELETE from job_models
    ├── Remove from JobsMap
    └── If no other jobs share this cronSpec:
        └── Remove cron entry from scheduler
```

---

## Files

| File | Purpose | Lines |
|------|---------|-------|
| models/cron_model.go | Job, JobPayload, JobModel structs | 37 |
| handlers/cron_handler.go | CRUD handlers | 201 |
| utils/cron_helper.go | Scheduler init, cron spec gen, job restore, global state | 139 |
| utils/sso_token_manager.go | SSO two-step login, token caching, thread-safe refresh | 298 |
| routes/cronRoutes.go | Route registration (no auth middleware) | 17 |

---

## Notes

- **No authentication on API** — `/cron/jobs` endpoints have no JWKS middleware (unlike other modules)
- **Table not in AutoMigrate** — `job_models` PostgreSQL table must exist already
- **Token shared per batch** — SSO token fetched once per cron tick, reused for all jobs in that tick
- **No request body support** — Jobs only fire HTTP requests with method + URL, no configurable body/headers (beyond SSO auth)
- **Sequential execution** — Jobs within a cron tick execute sequentially (not parallel goroutines)
