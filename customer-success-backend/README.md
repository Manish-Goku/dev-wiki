# Customer Success Backend

Unified omnichannel customer success portal. Consolidates emails, chats, and calls from multiple platforms into one place.

## Stack

| Component | Tech |
|-----------|------|
| Backend | NestJS 11, TypeScript (strict), port 3002 |
| Frontend | Vite + React + TypeScript + shadcn-ui, port 8080 |
| Database | Supabase (PostgreSQL), project: `gusenrddxwuwrclzfkur` |
| AI | Google Gemini 2.0 Flash |
| Real-time | Socket.io (WebSocket) + Supabase postgres_changes |
| Backend Repo | `git@github.com:Manish-Goku/CUSTOMER-SUCCESS-BACKEND.git` |
| Frontend Dir | `~/Desktop/katyayani-customer-success/` |

## Channels

| Channel | Platform | Integration | Status |
|---------|----------|-------------|--------|
| Email | Gmail | Gmail API + Pub/Sub (push) | `#wip` — code built, GCP setup pending |
| Chat | Interakt (WhatsApp) | Webhook ingestion + outbound API | `#active` |
| Chat | Netcore (WhatsApp) | Webhook ingestion + outbound API | `#active` |
| Calls | IVR | Frontend wired to Supabase `ivr_calls` (real-time) | `#active` |

## Modules — Backend

| Module | Path | Description | Tags |
|--------|------|-------------|------|
| Supabase | `src/supabase/` | Global DB client (service_role) | `#active` |
| Gmail Ingestion | `src/gmailIngestion/` | Gmail watch, webhook, CRUD, AI triage | `#wip` `#webhook` `#cron` |
| Email Gateway | `src/emailGateway/` | WebSocket gateway (namespace `/emails`) | `#active` |
| Chat Ingestion | `src/chatIngestion/` | Multi-provider WhatsApp ingestion + outbound | `#active` `#webhook` |
| Chat Gateway | `src/chatIngestion/chatGateway.ts` | WebSocket gateway (namespace `/chats`) | `#active` |

## Modules — Frontend (Hooks)

| Hook | Description | Real-time |
|------|-------------|-----------|
| `useIVRCalls` | Unified hook for all 7 IVR pages — filter, mutate, subscribe | Yes |
| `useIVRSidebarCounts` | Sidebar badges: IVR Live/Hangup per dept + SLA breach | Yes |
| `useLiveChat` | Conversations + messages from Supabase, send reply via backend | Yes |
| `useChatCounts` | Sidebar badges: Live Chat per channel | Yes |
| `useEmails` | Email listing with AI summaries | No |
| `useQueryAssignments` | CRUD + bulk assign for query assignments | No |
| `useAgentCalls`, `useAgentChats`, `useAgentHangups`, `useAgentSLABreach`, `useAgentCompleted` | Agent dashboard hooks | Yes |
| `useAllAgentsLive` | Agent status for department views | Yes |

## Models

- [support_emails](models/support_emails.md) — monitored mailboxes
- [emails](models/emails.md) — ingested email messages with AI triage
- [ivr_calls](models/ivr_calls.md) — IVR call records with SLA tracking

## Workflows

- [Gmail Ingestion Flow](workflows/gmail_ingestion.md) — end-to-end email ingestion pipeline
- [IVR Supabase Wiring](workflows/ivr_supabase_wiring.md) — IVR pages → Supabase data flow

## API Overview

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
| POST | `/webhooks/gmail` | Pub/Sub push endpoint | `#public-endpoint` `#webhook` |
| POST | `/webhooks/interakt` | Interakt WhatsApp webhook | `#public-endpoint` `#webhook` |
| POST | `/webhooks/netcore` | Netcore WhatsApp webhook | `#public-endpoint` `#webhook` |
| POST | `/conversations/:id/reply` | Send chat reply (routes to provider) | `#auth-required` |
| GET | `/chat-templates` | List canned responses | `#auth-required` |
| POST | `/chat-templates` | Create template | `#auth-required` |
| PATCH | `/chat-templates/:id` | Update template | `#auth-required` |
| DELETE | `/chat-templates/:id` | Delete template | `#auth-required` |
| GET | `/query-assignments` | List assignments | `#auth-required` |
| POST | `/query-assignments` | Create assignment | `#auth-required` |
| PATCH | `/query-assignments/:id` | Update assignment | `#auth-required` |
| POST | `/query-assignments/bulk-assign` | Bulk assign | `#auth-required` |

Swagger: `http://localhost:3002/api/docs`
