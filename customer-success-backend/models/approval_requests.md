# approval_requests

Approval workflow requests linked to notifications. Tracks who requested, status, reviewer, and comments.

## Table: `approval_requests`

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | uuid | NO | `gen_random_uuid()` | PK |
| `type` | text | NO | — | CHECK: `early_clockout`, `leave_request`, `ticket_escalation`, `refund_approval`, `call_transfer` |
| `requested_by` | text | NO | — | User ID of requester |
| `requested_by_name` | text | NO | — | Display name of requester |
| `requested_at` | timestamptz | YES | `now()` | When requested |
| `status` | text | NO | `'pending'` | CHECK: `pending`, `approved`, `rejected` |
| `reason` | text | NO | — | Reason for request |
| `metadata` | jsonb | YES | `'{}'` | Arbitrary context data (e.g. `{ orderId, amount, ticketId }`) |
| `reviewed_by` | text | YES | — | Who approved/rejected |
| `reviewed_at` | timestamptz | YES | — | When reviewed |
| `comments` | text | YES | — | Reviewer comments (required for rejection) |
| `notification_id` | uuid | YES | — | FK → `notifications.id` |
| `created_at` | timestamptz | YES | `now()` | |
| `updated_at` | timestamptz | YES | `now()` | |

## RLS
Disabled.

## Realtime
Enabled — frontend subscribes to `*` events for live approval status updates.

## Relationships
- `notification_id` → `notifications.id` (FK, one-to-one)

## Key behaviors
- Auto-created when a notification is created with `requires_approval=true`
- Approve endpoint (`POST /notifications/approval-requests/:id/approve`):
  - Sets `status='approved'`, `reviewed_by`, `reviewed_at`
  - Also updates linked `notifications` row: `approval_status='approved'`, `approved_by`, `approved_at`
- Reject endpoint (`POST /notifications/approval-requests/:id/reject`):
  - Sets `status='rejected'`, `reviewed_by`, `reviewed_at`, `comments`
  - Also updates linked `notifications` row: `approval_status='rejected'`, `rejected_reason`

## Frontend field mapping

| DB | Frontend (`ApprovalRequest` interface) |
|----|----------------------------------------|
| `requested_by` | `requestedBy` |
| `requested_by_name` | `requestedByName` |
| `requested_at` | `requestedAt` |
| `reviewed_by` | `reviewedBy` |
| `reviewed_at` | `reviewedAt` |
| `notification_id` | `notificationId` |
