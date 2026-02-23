# Customer Success Backend

Unified omnichannel customer success portal. Consolidates emails, chats, and calls from multiple platforms into one place.

## Stack

| Component | Tech |
|-----------|------|
| Framework | NestJS 11, TypeScript (strict) |
| Database | Supabase (PostgreSQL) |
| AI | Google Gemini 2.0 Flash |
| Real-time | Socket.io (WebSocket) |
| Port | 3002 |
| Repo | `git@github.com:Manish-Goku/CUSTOMER-SUCCESS-BACKEND.git` |

## Channels

| Channel | Platform | Integration | Status |
|---------|----------|-------------|--------|
| Email | Gmail | Gmail API + Pub/Sub (push) | `#wip` — code built, GCP setup pending |
| Chat | Interakt (WhatsApp) | TBD | `#needs-discussion` |
| Calls | IVR | TBD | `#needs-discussion` |

## Modules

| Module | Path | Description | Tags |
|--------|------|-------------|------|
| Supabase | `src/supabase/` | Global DB client (service_role) | `#active` |
| Gmail Ingestion | `src/gmailIngestion/` | Gmail watch, webhook, CRUD, AI triage | `#wip` `#webhook` `#cron` `#event-driven` |
| Email Gateway | `src/emailGateway/` | WebSocket gateway (namespace `/emails`) | `#active` |
| Dashboard | TBD | Admin analytics, daily stats | `#needs-discussion` |
| Agents | TBD | Support agent management + assignment | `#needs-discussion` |

## Models

- [support_emails](models/support_emails.md) — monitored mailboxes
- [emails](models/emails.md) — ingested email messages with AI triage

## Workflows

- [Gmail Ingestion Flow](workflows/gmail_ingestion.md) — end-to-end email ingestion pipeline

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

Swagger: `http://localhost:3002/api/docs`
