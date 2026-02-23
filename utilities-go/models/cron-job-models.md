# Cron Job Models — PostgreSQL (GORM)

`#active`

---

## JobModel

**Table:** `job_models` (GORM default naming, NOT in AutoMigrate — table created separately)
**File:** `models/cron_model.go`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| ID | int32 | PK | Auto-increment |
| URL | string | NOT NULL | HTTP endpoint to call |
| Method | string | NOT NULL, default: GET | HTTP method (GET, POST, etc.) |
| RepeatTime | string | NOT NULL | Go duration string (e.g., "30m", "2h", "24h") |
| StartTime | string | NOT NULL | Time of day in HH:MM format (e.g., "09:00", "14:30") |
| CronSpec | string | NOT NULL, indexed | Generated cron expression |
| CreatedAt | time.Time | Auto | |
| UpdatedAt | time.Time | Auto | |

### Cron Spec Generation

`RepeatTime` + `StartTime` are converted to a standard cron expression:

| RepeatTime | StartTime | Cron Spec | Meaning |
|-----------|-----------|-----------|---------|
| "30m" | any | `*/30 * * * *` | Every 30 minutes |
| "2h" | "14:30" | `30 */2 * * *` | At :30 past every 2 hours |
| "24h" | "09:00" | `0 9 * * *` | Daily at 09:00 |

---

## Job (in-memory runtime)

**File:** `models/cron_model.go`

Same fields as JobModel but used in the runtime `JobsMap`. Not persisted — rebuilt from DB on startup via `RestoreJobs()`.

---

## JobPayload (request body)

**File:** `models/cron_model.go`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| url | string | Yes | HTTP endpoint |
| method | string | Yes | HTTP method |
| repeatTime | string | Yes | Go duration (e.g., "1h") |
| startTime | string | Yes | HH:MM format |

---

## Global Runtime State

**File:** `utils/cron_helper.go`

| Variable | Type | Purpose |
|----------|------|---------|
| `JobsMap` | `map[int32]*Job` | All active jobs keyed by ID |
| `CronEntriesMap` | `map[string]cron.EntryID` | Cron spec → scheduler entry (deduped) |
| `Mu` | `sync.Mutex` | Protects JobsMap |
| `CronScheduler` | `*cron.Cron` | robfig/cron scheduler instance |

Multiple jobs can share the same cron spec — only one cron entry is registered for each unique spec.
