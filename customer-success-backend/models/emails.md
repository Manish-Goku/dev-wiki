# emails

Ingested email messages from monitored mailboxes. Includes AI-generated summary and team classification.

## Schema

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| id | UUID | `gen_random_uuid()` | PK |
| support_email_id | UUID | — | FK → support_emails.id (CASCADE) |
| gmail_message_id | TEXT | — | Gmail message ID (UNIQUE, dedup key) |
| thread_id | TEXT | NULL | Gmail thread ID |
| history_id | TEXT | NULL | Gmail history ID |
| from_address | TEXT | — | Sender email |
| from_name | TEXT | NULL | Sender display name |
| to_addresses | TEXT[] | `{}` | Recipients |
| cc_addresses | TEXT[] | `{}` | CC recipients |
| bcc_addresses | TEXT[] | `{}` | BCC recipients |
| subject | TEXT | NULL | Email subject |
| body_text | TEXT | NULL | Plain text body |
| body_html | TEXT | NULL | HTML body |
| snippet | TEXT | NULL | Gmail snippet |
| label_ids | TEXT[] | `{}` | Gmail labels |
| has_attachments | BOOLEAN | false | |
| attachments | JSONB | `[]` | Metadata: `[{attachment_id, filename, mime_type, size}]` |
| internal_date | TIMESTAMPTZ | NULL | Gmail internal date |
| received_at | TIMESTAMPTZ | `now()` | When ingested |
| is_read | BOOLEAN | false | |
| summary | TEXT | NULL | AI-generated summary (Gemini) |
| suggested_team | TEXT | NULL | AI-suggested team: finance, support, dispatch, sales, technical, returns_refunds, general |
| created_at | TIMESTAMPTZ | `now()` | |

## Indexes

| Name | Columns |
|------|---------|
| idx_emails_support_email | support_email_id |
| idx_emails_gmail_message | gmail_message_id |
| idx_emails_thread | thread_id |
| idx_emails_from | from_address |
| idx_emails_received | received_at DESC |
| idx_emails_suggested_team | suggested_team |

## AI Triage

On ingestion, Gemini 2.0 Flash analyzes subject + body and produces:
- **summary** — 1-2 sentence description of customer intent
- **suggested_team** — one of: `finance`, `support`, `dispatch`, `sales`, `technical`, `returns_refunds`, `general`

Fallback on AI failure: summary = subject, team = general.
