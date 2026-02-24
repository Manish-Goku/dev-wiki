# notification_preferences

Per-user notification channel preferences. Controls which notification types trigger email, SMS, or push alerts.

## Table: `notification_preferences`

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `user_id` | text | NO | — | User identifier (e.g. `'manish'`) |
| `name` | text | NO | — | Preference name (e.g. `'SLA Breach Alerts'`) |
| `description` | text | YES | — | Human-readable description |
| `email` | boolean | NO | `true` | Receive via email |
| `sms` | boolean | NO | `false` | Receive via SMS |
| `push` | boolean | NO | `true` | Receive via push notification |
| `created_at` | timestamptz | YES | `now()` | |
| `updated_at` | timestamptz | YES | `now()` | |

## RLS
Disabled.

## Realtime
Not subscribed (preferences are fetched on page load, persisted via debounced PUT).

## Key behaviors
- **Auto-seeding**: On first `GET /notifications/preferences/:user_id`, if no rows exist for that user, backend auto-creates 6 default preferences:
  1. SLA Breach Alerts (email: true, sms: true, push: true)
  2. Escalation Notifications (email: true, sms: false, push: true)
  3. New Ticket Assignment (email: true, sms: false, push: true)
  4. Missed Call Alerts (email: true, sms: true, push: true)
  5. Daily Performance Summary (email: true, sms: false, push: false)
  6. VIP Customer Alerts (email: true, sms: true, push: true)
- **Upsert**: `PUT /notifications/preferences/:user_id` receives full array of preferences, upserts by `user_id + name`

## Frontend integration
- **Settings.tsx** (Notifications tab): Fetched on mount via `GET /notifications/preferences/manish`
- Toggles update local state + debounced `PUT` (500ms delay)
- Fallback: if fetch fails, shows `DEFAULT_NOTIFICATION_SETTINGS` hardcoded array
