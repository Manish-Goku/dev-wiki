# emails

Email messages (inbound + outbound) for monitored mailboxes. Inbound emails include AI-generated summary and team classification. Outbound replies track which agent sent them.

## Schema

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| id | UUID | `gen_random_uuid()` | PK |
| support_email_id | UUID | — | FK → support_emails.id (CASCADE) |
| message_id | TEXT | — | Email Message-ID header (UNIQUE, dedup key) |
| thread_id | TEXT | NULL | Threading key — for outbound replies, set to original's message_id |
| from_address | TEXT | — | Sender email |
| from_name | TEXT | NULL | Sender display name |
| to_addresses | TEXT[] | `{}` | Recipients |
| cc_addresses | TEXT[] | `{}` | CC recipients |
| bcc_addresses | TEXT[] | `{}` | BCC recipients |
| subject | TEXT | NULL | Email subject |
| body_text | TEXT | NULL | Plain text body |
| body_html | TEXT | NULL | HTML body |
| snippet | TEXT | NULL | Preview snippet |
| has_attachments | BOOLEAN | false | |
| attachments | JSONB | `[]` | Metadata: `[{attachment_id, filename, mime_type, size}]` |
| internal_date | TIMESTAMPTZ | NULL | Original send date |
| received_at | TIMESTAMPTZ | `now()` | When ingested/sent |
| is_read | BOOLEAN | false | |
| summary | TEXT | NULL | AI-generated summary (Gemini) |
| suggested_team | TEXT | NULL | AI-suggested team: finance, support, dispatch, sales, technical, returns_refunds, general |
| **direction** | TEXT NOT NULL | `'inbound'` | `'inbound'` (customer → us) or `'outbound'` (agent → customer) |
| **agent_name** | TEXT | NULL | Agent who sent the reply (outbound only) |
| **in_reply_to** | TEXT | NULL | message_id of the email being replied to (outbound only) |
| created_at | TIMESTAMPTZ | `now()` | |

## Indexes

| Name | Columns | Notes |
|------|---------|-------|
| idx_emails_support_email | support_email_id | |
| idx_emails_thread | thread_id | |
| idx_emails_from | from_address | |
| idx_emails_received | received_at DESC | |
| idx_emails_suggested_team | suggested_team | |
| idx_emails_in_reply_to | in_reply_to | Partial: WHERE in_reply_to IS NOT NULL |

## Thread Grouping

Inbound and outbound emails are linked by `thread_id`:
- **Inbound:** `thread_id` may be NULL; `message_id` is the unique identifier
- **Outbound reply:** `thread_id` = original inbound's `message_id`, `in_reply_to` = same

Frontend groups rows by `thread_id || message_id` to build conversation threads.

## AI Triage (inbound only)

On ingestion, Gemini 2.0 Flash analyzes subject + body and produces:
- **summary** — 1-2 sentence description of customer intent
- **suggested_team** — one of: `finance`, `support`, `dispatch`, `sales`, `technical`, `returns_refunds`, `general`

Fallback on AI failure: summary = subject, team = general.
