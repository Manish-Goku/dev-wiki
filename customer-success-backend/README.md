# Customer Success Backend

Unified omnichannel customer success portal. Consolidates emails, chats, and calls from multiple platforms into one place.

## Stack

| Component | Tech |
|-----------|------|
| Backend | NestJS 10, TypeScript (strict), port **3002** |
| Frontend | Vite + React + TypeScript + shadcn-ui, port **8080** |
| Database | Supabase (PostgreSQL), project: `gusenrddxwuwrclzfkur` (ap-south-1) |
| AI | Google Gemini 2.0 Flash |
| Real-time | Socket.io (WebSocket) + Supabase postgres_changes |
| Backend Repo | `git@github.com:Manish-Goku/CUSTOMER-SUCCESS-BACKEND.git` |
| Backend Dir | `~/Desktop/CUSTOMER-SUCCESS/customer-success-backend/` |
| Frontend Dir | `~/Desktop/CUSTOMER-SUCCESS/katyayani-customer-success/` |

## How to Run

```bash
# Backend (port 3002)
cd ~/Desktop/CUSTOMER-SUCCESS/customer-success-backend
npm run start:dev        # nest start --watch (hot reload)

# Frontend (port 8080)
cd ~/Desktop/CUSTOMER-SUCCESS/katyayani-customer-success
npm run dev              # vite dev server

# Access
# Frontend: http://localhost:8080
# Backend API: http://localhost:3002
# Swagger docs: http://localhost:3002/api/docs
```

## Channels

| Channel | Platform | Integration | Status |
|---------|----------|-------------|--------|
| Email | Gmail | IMAP polling + SMTP reply | `#active` — inbound + outbound working |
| Chat | Interakt (WhatsApp) | Webhook ingestion + outbound API | `#active` |
| Chat | Netcore (WhatsApp) | Webhook ingestion + outbound API | `#active` |
| Calls | IVR | Dynamic providers via `ivr_providers` + webhook ingestion | `#active` |
| Calls | Kaleyra Voice | Click-to-call + call sync + dashboard analytics | `#active` |

## Modules — Backend

| Module | Path | Description | Tags |
|--------|------|-------------|------|
| Supabase | `src/supabase/` | Global DB client (service_role) | `#active` |
| Gmail Ingestion | `src/gmailIngestion/` | IMAP polling, SMTP reply, CRUD, AI triage | `#active` `#smtp` `#cron` |
| Email Gateway | `src/emailGateway/` | WebSocket gateway (namespace `/emails`) | `#active` |
| Chat Ingestion | `src/chatIngestion/` | Multi-provider WhatsApp ingestion + outbound | `#active` `#webhook` |
| Chat Gateway | `src/chatIngestion/chatGateway.ts` | WebSocket gateway (namespace `/chats`) | `#active` |
| IVR Webhooks | `src/ivrWebhooks/` | Dynamic IVR provider webhooks — field mapping + status mapping | `#active` `#webhook` |
| Kaleyra Voice | `src/kaleyraVoice/` | Click-to-call, outbound calls, Kaleyra→Supabase sync, call dashboard | `#active` `#api-sync` |
| Admin Dashboard | `src/adminDashboard/` | Email analytics dashboard (volume, teams, senders, daily trend) | `#active` |
| Notifications | `src/notifications/` | Notifications + approval requests + preferences. Cross-module injectable | `#active` |
| Agents | `src/agents/` | Agent CRUD, clock-in/out, daily stats | `#active` |
| SLA Config | `src/slaConfig/` | SLA tier configuration (vip/high/normal/low) | `#active` |
| Roles | `src/roles/` | Role-based access, module permissions, user-role assignments | `#active` |
| Refunds | `src/refunds/` | Refund request lifecycle, actions, stats | `#active` |
| Audits | `src/audits/` | Interaction audit scoring and tracking | `#active` |
| Video Call Leads | `src/videoCallLeads/` | Video call lead management | `#active` |
| Call Categories | `src/callCategories/` | IVR disposition categories (tree structure) | `#active` |
| Customer Timeline | `src/customerTimeline/` | Cross-channel customer event timeline | `#active` |
| Tickets | `src/tickets/` | Ticket system (auto-gen TKT-NNNN) | `#active` |
| Surveys | `src/surveys/` | Customer satisfaction surveys | `#active` |
| Agent Feedback | `src/agentFeedback/` | Agent performance feedback | `#active` |
| Workflows | `src/workflows/` | Workflow automation engine | `#active` |
| Youtube Leads | `src/youtubeLeads/` | YouTube comment lead capture (auto-gen YT-NNNN) | `#active` |
| Customer Feedback | `src/customerFeedback/` | Customer feedback collection | `#active` |
| Return RCA | `src/returnRca/` | Return root cause analysis | `#active` |
| Agri Consultancy | `src/agriConsultancy/` | Agriculture consultancy management | `#active` |

## Modules — Frontend (Hooks)

| Hook | Description | Real-time |
|------|-------------|-----------|
| `useIVRCalls` | Unified hook for all 7 IVR pages — filter, mutate, subscribe | Yes |
| `useIVRProviders` | Dynamic provider registry — CRUD, labels, realtime subscription | Yes |
| `useCallCategories` | Call categories per department from `call_categories` table | No |
| `useIVRSidebarCounts` | Sidebar badges: IVR Live/Hangup per dept (dynamic) + SLA breach | Yes |
| `useLiveChat` | Conversations + messages from Supabase, send reply via backend | Yes |
| `useChatCounts` | Sidebar badges: Live Chat per channel | Yes |
| `useEmails` | Email listing, thread grouping, send reply via backend, AI summaries | Yes |
| `useQueryAssignments` | CRUD + bulk assign for query assignments | No |
| `useAgentCalls`, `useAgentChats`, `useAgentHangups`, `useAgentSLABreach`, `useAgentCompleted` | Agent dashboard hooks | Yes |
| `useAllAgentsLive` | Agent status for department views | Yes |
| `useNotifications` | Notifications + approval requests, backend mutations, realtime, computed stats | Yes |
| `useKaleyraDashboard` | **PENDING** — call dashboard overview, agent stats, daily volume, sync | No |
| `useKaleyraCallList` | **PENDING** — paginated call log with filters | No |

## Models

- [support_emails](models/support_emails.md) — monitored mailboxes
- [emails](models/emails.md) — ingested email messages with AI triage
- [ivr_calls](models/ivr_calls.md) — IVR call records with SLA tracking
- [ivr_providers](models/ivr_providers.md) — Dynamic IVR provider registry (webhook config, field mapping)
- [kaleyra_call_logs](models/kaleyra_call_logs.md) — synced Kaleyra call data with location, provider, recording, status
- [notifications](models/notifications.md) — main notification store with approval support
- [approval_requests](models/approval_requests.md) — approval requests linked to notifications
- [notification_preferences](models/notification_preferences.md) — per-user notification channel preferences

## Workflows

- [Gmail Ingestion Flow](workflows/gmail_ingestion.md) — end-to-end email ingestion pipeline
- [Email Reply Pipeline](workflows/email_reply_pipeline.md) — SMTP send, DB persist, thread grouping
- [IVR Supabase Wiring](workflows/ivr_supabase_wiring.md) — IVR pages → Supabase data flow
- [Dynamic IVR Providers](workflows/dynamic_ivr_providers.md) — Provider registration, webhook ingestion, auto-rendering
- [Notifications Wiring](workflows/notifications_wiring.md) — End-to-end notifications + approvals + preferences flow
- [Kaleyra Call Dashboard](workflows/kaleyra_call_dashboard.md) — Kaleyra API sync, call analytics, agent stats, dashboard endpoints

## API Overview

### Email

| Method | Path | Description | Tags |
|--------|------|-------------|------|
| POST | `/support-emails` | Add email to monitor | `#auth-required` |
| GET | `/support-emails` | List monitored emails | `#auth-required` |
| GET | `/support-emails/:id` | Get single | `#auth-required` |
| PATCH | `/support-emails/:id` | Update (toggle active, name) | `#auth-required` |
| DELETE | `/support-emails/:id` | Remove + stop watch | `#auth-required` |
| POST | `/support-emails/:id/watch` | Manual watch start | `#auth-required` |
| POST | `/support-emails/:id/stop-watch` | Manual watch stop | `#auth-required` |
| GET | `/emails` | List emails (paginated) | `#auth-required` |
| GET | `/emails/:id` | Full email + AI summary | `#auth-required` |
| POST | `/emails/:id/reply` | Send SMTP reply (with CC/BCC) | `#auth-required` `#smtp` |
| POST | `/webhooks/gmail` | Pub/Sub push endpoint | `#public-endpoint` `#webhook` |

### Chat

| Method | Path | Description | Tags |
|--------|------|-------------|------|
| POST | `/webhooks/interakt` | Interakt WhatsApp webhook (legacy) | `#public-endpoint` `#webhook` |
| POST | `/webhooks/interact/:slug` | Dynamic Interact provider webhook | `#public-endpoint` `#webhook` |
| POST | `/webhooks/netcore` | Netcore WhatsApp webhook | `#public-endpoint` `#webhook` |
| POST | `/conversations/:id/reply` | Send chat reply (routes to provider) | `#auth-required` |
| GET | `/chat-templates` | List canned responses | `#auth-required` |
| POST | `/chat-templates` | Create template | `#auth-required` |
| PATCH | `/chat-templates/:id` | Update template | `#auth-required` |
| DELETE | `/chat-templates/:id` | Delete template | `#auth-required` |

### IVR & Calls

| Method | Path | Description | Tags |
|--------|------|-------------|------|
| POST | `/webhooks/ivr/:slug` | IVR provider webhook (fire-and-forget) | `#public-endpoint` `#webhook` |
| POST | `/calls/click-to-call` | Kaleyra click-to-call (agent→customer bridge) | `#auth-required` |
| POST | `/calls/outbound` | Kaleyra automated outbound call (OBD) | `#auth-required` |
| GET | `/webhooks/kaleyra/callback` | Kaleyra callback (GET with query params) | `#public-endpoint` `#webhook` |
| GET | `/calls/logs` | Raw call logs from Kaleyra API | `#auth-required` |
| POST | `/calls/sync` | Sync Kaleyra call logs → Supabase | `#auth-required` |
| GET | `/calls/dashboard/overview` | All call metrics (volume, agents, daily, status) | `#auth-required` |
| GET | `/calls/dashboard/agent-stats` | Per-agent: answered, talk time, missed | `#auth-required` |
| GET | `/calls/dashboard/daily-volume` | Daily incoming + forwarded trend (IST) | `#auth-required` |
| GET | `/calls/dashboard/call-list` | Paginated call log (filters: service, agent, date) | `#auth-required` |

### Assignments & Notifications

| Method | Path | Description | Tags |
|--------|------|-------------|------|
| GET | `/query-assignments` | List assignments | `#auth-required` |
| POST | `/query-assignments` | Create assignment | `#auth-required` |
| PATCH | `/query-assignments/:id` | Update assignment | `#auth-required` |
| POST | `/query-assignments/bulk-assign` | Bulk assign | `#auth-required` |
| POST | `/notifications` | Create notification (optionally auto-creates approval_request) | `#auth-required` |
| GET | `/notifications` | List with filters (type, status, priority, to_user, search, page, limit) | `#auth-required` |
| GET | `/notifications/unread-count/:user_id` | Unread badge count | `#auth-required` |
| PATCH | `/notifications/mark-all-read` | Bulk mark all unread → read | `#auth-required` |
| PATCH | `/notifications/:id` | Mark single read/actioned/dismissed | `#auth-required` |
| GET | `/notifications/approval-requests` | List approval requests (with filters) | `#auth-required` |
| POST | `/notifications/approval-requests/:id/approve` | Approve request (updates both tables) | `#auth-required` |
| POST | `/notifications/approval-requests/:id/reject` | Reject with reason (updates both tables) | `#auth-required` |
| GET | `/notifications/preferences/:user_id` | Get preferences (auto-seeds 6 defaults) | `#auth-required` |
| PUT | `/notifications/preferences/:user_id` | Upsert preferences | `#auth-required` |

### Admin Dashboard

| Method | Path | Description | Tags |
|--------|------|-------------|------|
| GET | `/admin/dashboard/overview` | All email dashboard metrics (parallel) | `#auth-required` |
| GET | `/admin/dashboard/email-volume` | Email count in date range | `#auth-required` |
| GET | `/admin/dashboard/emails-by-team` | Email distribution by team | `#auth-required` |
| GET | `/admin/dashboard/unread-count` | Unread email count | `#auth-required` |
| GET | `/admin/dashboard/top-senders` | Top email senders | `#auth-required` |
| GET | `/admin/dashboard/daily-volume` | Daily email volume trend | `#auth-required` |
| GET | `/admin/dashboard/emails-by-mailbox` | Emails per monitored mailbox | `#auth-required` |

### Agents & Config

| Method | Path | Description | Tags |
|--------|------|-------------|------|
| POST | `/agents` | Create agent | `#auth-required` |
| GET | `/agents` | List with filters | `#auth-required` |
| GET | `/agents/:id` | Get agent | `#auth-required` |
| PATCH | `/agents/:id` | Update agent | `#auth-required` |
| DELETE | `/agents/:id` | Soft delete | `#auth-required` |
| PATCH | `/agents/:id/status` | Quick status toggle | `#auth-required` |
| PATCH | `/agents/:id/clock-in` | Clock in | `#auth-required` |
| PATCH | `/agents/:id/clock-out` | Clock out | `#auth-required` |
| GET | `/agents/:id/stats` | Last 7 days daily stats | `#auth-required` |
| GET | `/sla-config` | List SLA tier configs | `#auth-required` |
| PUT | `/sla-config/:tier` | Update tier | `#auth-required` |
| PUT | `/sla-config/bulk` | Bulk update all tiers | `#auth-required` |
| GET | `/roles` | List roles | `#auth-required` |
| POST | `/roles` | Create role | `#auth-required` |
| GET | `/roles/modules` | List all modules | `#auth-required` |
| GET | `/roles/:id/modules` | Modules for role | `#auth-required` |
| PUT | `/roles/:id/modules` | Set module access | `#auth-required` |
| GET | `/roles/users` | List user-role assignments | `#auth-required` |
| POST | `/roles/users` | Assign role | `#auth-required` |
| DELETE | `/roles/users/:id` | Remove assignment | `#auth-required` |

### Operations

| Method | Path | Description | Tags |
|--------|------|-------------|------|
| POST | `/refunds` | Create refund request | `#auth-required` |
| GET | `/refunds` | List with filters | `#auth-required` |
| GET | `/refunds/:id` | Get with products + history | `#auth-required` |
| PATCH | `/refunds/:id` | Update status/assignment | `#auth-required` |
| POST | `/refunds/:id/actions` | Add action (approve/reject/comment) | `#auth-required` |
| GET | `/refunds/stats` | Summary stats | `#auth-required` |
| GET | `/audits` | List audits | `#auth-required` |
| GET | `/audits/:id` | Get audit | `#auth-required` |
| PATCH | `/audits/:id` | Update audit | `#auth-required` |
| GET | `/audits/stats` | Audit summary | `#auth-required` |
| POST | `/video-call-leads` | Create lead | `#auth-required` |
| GET | `/video-call-leads` | List with filters | `#auth-required` |
| GET | `/video-call-leads/:id` | Get lead | `#auth-required` |
| PATCH | `/video-call-leads/:id` | Update lead | `#auth-required` |
| DELETE | `/video-call-leads/:id` | Delete lead | `#auth-required` |
| GET | `/video-call-leads/stats` | Lead stats | `#auth-required` |
| GET | `/call-categories` | List all (tree) | `#auth-required` |
| GET | `/call-categories/by-department/:dept` | By department | `#auth-required` |
| POST | `/call-categories` | Create | `#auth-required` |
| PATCH | `/call-categories/:id` | Update | `#auth-required` |
| DELETE | `/call-categories/:id` | Soft delete | `#auth-required` |
| GET | `/customer-timeline/:mobile_number` | All events for customer | `#auth-required` |
| GET | `/customer-timeline/:mobile_number/summary` | Event counts | `#auth-required` |
| POST | `/customer-timeline` | Create event | `#auth-required` |

### Phase 2 Modules

| Method | Path | Description | Tags |
|--------|------|-------------|------|
| POST | `/tickets` | Create ticket (auto TKT-NNNN) | `#auth-required` |
| GET | `/tickets` | List with filters | `#auth-required` |
| GET | `/tickets/:id` | Get ticket | `#auth-required` |
| PATCH | `/tickets/:id` | Update ticket | `#auth-required` |
| GET | `/tickets/stats` | Ticket stats | `#auth-required` |
| POST | `/surveys` | Create survey | `#auth-required` |
| GET | `/surveys` | List surveys | `#auth-required` |
| POST | `/agent-feedback` | Submit feedback | `#auth-required` |
| GET | `/agent-feedback` | List feedback | `#auth-required` |
| POST | `/workflows` | Create workflow | `#auth-required` |
| GET | `/workflows` | List workflows | `#auth-required` |
| POST | `/youtube-leads` | Create lead (auto YT-NNNN) | `#auth-required` |
| GET | `/youtube-leads` | List leads | `#auth-required` |
| POST | `/customer-feedback` | Submit feedback | `#auth-required` |
| GET | `/customer-feedback` | List feedback | `#auth-required` |
| POST | `/return-rca` | Create RCA | `#auth-required` |
| GET | `/return-rca` | List RCAs | `#auth-required` |
| POST | `/agri-consultancy` | Create consultation | `#auth-required` |
| GET | `/agri-consultancy` | List consultations | `#auth-required` |

**Total: 160+ endpoints across 26 modules.**

Swagger: `http://localhost:3002/api/docs`

## Kaleyra Voice — IVR Setup

| Item | Value |
|------|-------|
| IVR DID Number | 8068921234 |
| API Base | `https://api-voice.kaleyra.com/v1/` |
| Auth | `api_key` form param (env: `KALEYRA_API_KEY`) |
| Callback | GET with URL template placeholders (env: `KALEYRA_CALLBACK_URL`) |
| Agent Numbers | 9201952685, 9201972062, 6264437618, 9201836530, 9238109518, 9201972072 |
| Call Flow | Customer → Incoming (DID) → CallForward chain (agents sequentially) |
| Recording Host | `play.solutionsinfini.com` |
| Sync Method | `method=dial` with fromdate/todate pagination |
