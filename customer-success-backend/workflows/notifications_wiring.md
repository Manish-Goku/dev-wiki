# Notifications Wiring — End-to-End Flow

Complete flow for notifications, approval requests, and preferences from backend to frontend.

## Architecture

```
Backend (NestJS)                     Supabase                      Frontend (React)
┌──────────────┐                ┌──────────────────┐         ┌─────────────────────────┐
│ Notifications │  write via    │ notifications     │  read   │ useNotifications hook    │
│ Service       │──────────────>│ approval_requests │<────────│   - fetchNotifications() │
│ (10 methods)  │  service_role │ notification_prefs│         │   - fetchApprovalReqs()  │
└──────────────┘                └────────┬─────────┘         │   - realtime subscribe   │
       │                                 │                   └────────┬────────────────┘
       │                                 │ postgres_changes           │
       │                                 │ (INSERT/UPDATE/DELETE)     │
       │                                 └───────────────────────────>│ auto-refetch
       │                                                              │
       │  mutations via HTTP                                          │
       │<─────────────────────────────────────────────────────────────│
       │  PATCH /notifications/:id                                    │
       │  PATCH /notifications/mark-all-read                         │
       │  POST  /notifications/approval-requests/:id/approve         │
       │  POST  /notifications/approval-requests/:id/reject          │
       │  GET   /notifications/preferences/:user_id                  │
       │  PUT   /notifications/preferences/:user_id                  │
```

## Data flow

### 1. Notification creation
- Backend service (or any module importing `NotificationsService`) calls `create_notification(dto)`
- If `requires_approval=true`: auto-creates linked `approval_requests` row with `notification_id` FK
- Row written to Supabase via service_role client
- Supabase Realtime broadcasts INSERT event

### 2. Frontend receives notification
- `useNotifications` hook subscribes to `notifications` + `approval_requests` tables via `supabase.channel('notifications-realtime')`
- On any `*` event → refetch full lists via Supabase client (anon key)
- Mapped from snake_case DB fields to camelCase frontend interfaces
- Computed stats updated: `unreadCount`, `slaCount`, `missedCallCount`, `pendingApprovals`

### 3. Notification actions (mutations via backend API)
All mutations go through backend HTTP endpoints (not direct Supabase writes):

| Action | Frontend call | Backend endpoint | What happens |
|--------|--------------|-----------------|-------------|
| Mark read | `markRead(id)` | `PATCH /notifications/:id` `{status:'read'}` | Updates `status`, sets `read_at` |
| Mark dismissed | `markDismissed(id)` | `PATCH /notifications/:id` `{status:'dismissed'}` | Updates `status` |
| Mark all read | `markAllRead(toUser)` | `PATCH /notifications/mark-all-read` `{to_user}` | Bulk update all unread for user |
| Approve | `approveRequest(id, reviewedBy, comments?)` | `POST .../approve` | Updates `approval_requests` + linked `notifications` |
| Reject | `rejectRequest(id, reviewedBy, comments)` | `POST .../reject` | Updates `approval_requests` + linked `notifications` |

After each mutation, Supabase Realtime fires UPDATE → hook auto-refetches → UI updates.

### 4. Preferences flow
- **On mount** (Settings page): `GET /notifications/preferences/manish` → backend auto-seeds 6 defaults if none exist
- **On toggle**: local state updates immediately → debounced `PUT` (500ms) persists to backend
- Backend upserts by `user_id + name` combination

## Frontend consumers

### `src/hooks/useNotifications.ts`
The central hook. Used by 4 components:

| Consumer | What it uses |
|----------|-------------|
| `Notifications.tsx` | Full hook: `notifications`, `approvalRequests`, `unreadCount`, `loading`, `slaCount`, `missedCallCount`, `pendingApprovals`, all mutations, `refresh()` |
| `AppHeader.tsx` | `notifications` (latest 3 unread for dropdown), `unreadCount` (badge) |
| `AppSidebar.tsx` | `unreadCount` (nav badge) |
| `Settings.tsx` | Does NOT use hook — fetches preferences directly via `fetch()` to `GET/PUT` endpoints |

### State shape
```typescript
{
  notifications: Notification[]        // from Supabase, mapped to camelCase
  approvalRequests: ApprovalRequest[]  // from Supabase, mapped to camelCase
  unreadCount: number                  // notifications.filter(n => n.status === 'unread').length
  loading: boolean
  slaCount: number                     // useMemo — notifications.filter(n => n.type === 'sla_breach').length
  missedCallCount: number              // useMemo — notifications.filter(n => n.type === 'missed_call').length
  pendingApprovals: number             // useMemo — approvalRequests.filter(a => a.status === 'pending').length
}
```

## Supabase types
All 3 tables added to `src/integrations/supabase/types.ts` under `Tables`:
- `notifications` — Row, Insert, Update, Relationships (empty)
- `approval_requests` — Row, Insert, Update, Relationships (FK to notifications)
- `notification_preferences` — Row, Insert, Update, Relationships (empty)

**Gotcha**: New Supabase tables MUST be added to types.ts manually — typed client silently fails for unknown tables.

## Files involved

| File | Role |
|------|------|
| `src/integrations/supabase/types.ts` | Type definitions for 3 tables |
| `src/hooks/useNotifications.ts` | Central hook — fetches, mutations, realtime, computed stats |
| `src/data/notificationData.ts` | Type interfaces (`Notification`, `ApprovalRequest`) — still used for types only, mock data no longer consumed |
| `src/pages/Notifications.tsx` | Main page — 5 tabs, search, approve/reject dialog, stats cards |
| `src/components/layout/AppHeader.tsx` | Bell icon badge + dropdown (latest 3 unread) |
| `src/components/layout/AppSidebar.tsx` | Nav badge for Notifications item |
| `src/pages/Settings.tsx` | Notification preferences tab — fetch + debounced persist |

## Testing

```bash
# Create a notification
curl -X POST http://localhost:3002/notifications \
  -H 'Content-Type: application/json' \
  -d '{"type":"missed_call","title":"Missed Call","message":"Call from 9876543210","priority":"high","to_user":"manish"}'

# Create approval notification
curl -X POST http://localhost:3002/notifications \
  -H 'Content-Type: application/json' \
  -d '{"type":"approval_request","title":"Early Clock Out","message":"Rishi wants to leave early","priority":"high","to_user":"manish","requires_approval":true,"from_user":"rishi","metadata":{"requested_by":"rishi","requested_by_name":"Rishi Kumar","approval_type":"early_clockout","reason":"Personal emergency"}}'

# Mark all read
curl -X PATCH http://localhost:3002/notifications/mark-all-read \
  -H 'Content-Type: application/json' \
  -d '{"to_user":"manish"}'

# Get preferences (auto-seeds 6 defaults)
curl http://localhost:3002/notifications/preferences/manish
```

## Verification checklist
1. Open Notifications page → shows empty state initially
2. Create notification via curl → appears in real-time
3. Bell icon badge updates, sidebar badge updates
4. Create approval → appears in Approvals tab + pending panel
5. Approve/reject → updates both tables, UI reflects
6. Mark all read → all become read, badges go to 0
7. Settings → preferences load from DB, toggles persist
