# notifications

Main notification store. Supports approval workflows, SLA breach alerts, missed calls, assignments, escalations, feedback, and system events.

## Table: `notifications`

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `type` | text | NO | — | CHECK: `approval_request`, `sla_breach`, `missed_call`, `escalation`, `assignment`, `system`, `feedback`, `early_clockout` |
| `title` | text | NO | — | Short display title |
| `message` | text | NO | — | Full notification body |
| `priority` | text | NO | `'normal'` | CHECK: `low`, `normal`, `high`, `urgent` |
| `status` | text | NO | `'unread'` | CHECK: `unread`, `read`, `actioned`, `dismissed` |
| `reference_id` | text | YES | — | ID of related entity (ticket, call, etc.) |
| `reference_type` | text | YES | — | CHECK: `call`, `ticket`, `chat`, `agent`, `workflow` |
| `from_user` | text | YES | — | User/agent who triggered the notification |
| `to_user` | text | NO | — | Target user for the notification |
| `department` | text | YES | — | Department context |
| `metadata` | jsonb | YES | `'{}'` | Arbitrary key-value data |
| `requires_approval` | boolean | YES | `false` | If true, linked `approval_requests` row auto-created |
| `approval_status` | text | YES | — | CHECK: `pending`, `approved`, `rejected` |
| `approved_by` | text | YES | — | Who approved/rejected |
| `approved_at` | timestamptz | YES | — | When approved |
| `rejected_reason` | text | YES | — | Rejection reason (synced from approval_requests) |
| `read_at` | timestamptz | YES | — | When marked read |
| `actioned_at` | timestamptz | YES | — | When actioned |
| `created_at` | timestamptz | YES | `now()` | |
| `updated_at` | timestamptz | YES | `now()` | |

## RLS
Disabled (public access via anon key for frontend reads).

## Realtime
Enabled — frontend subscribes to `*` events on this table for live updates.

## Relationships
- `approval_requests.notification_id` → `notifications.id` (one-to-one FK)

## Key behaviors
- When `requires_approval=true` on creation, backend auto-creates linked `approval_requests` row
- Approve/reject on `approval_requests` also updates `approval_status`, `approved_by`, `approved_at`, `rejected_reason` on this table
- `mark-all-read` endpoint updates all `status='unread'` rows for a given `to_user`

## Frontend field mapping (snake_case → camelCase)

| DB | Frontend (`Notification` interface) |
|----|-------------------------------------|
| `created_at` | `createdAt` |
| `read_at` | `readAt` |
| `actioned_at` | `actionedAt` |
| `reference_id` | `referenceId` |
| `reference_type` | `referenceType` |
| `from_user` | `fromUser` |
| `to_user` | `toUser` |
| `requires_approval` | `requiresApproval` |
| `approval_status` | `approvalStatus` |
| `approved_by` | `approvedBy` |
| `approved_at` | `approvedAt` |
| `rejected_reason` | `rejectedReason` |
