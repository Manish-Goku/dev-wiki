# Email Reply Pipeline

`#active` `#smtp` `#threading`

## Overview

Full outbound email reply pipeline: agent sends reply from frontend → NestJS backend sends via SMTP (Gmail) → persists in DB with `direction = 'outbound'` → frontend groups into thread.

## Flow

```
1. Agent opens email in full-page view → clicks Reply → types message
   └→ Optional: toggle CC/BCC fields, add comma-separated addresses

2. Frontend calls POST /emails/:id/reply
   └→ body: { body_text, body_html?, agent_name?, cc?, bcc? }

3. Backend (GmailIngestionService.send_reply):
   a. Fetch original email row from `emails` table
   b. Fetch support_email row → decrypt IMAP/SMTP password (AES-256-GCM)
   c. Build SMTP envelope:
      - from: support email (display_name if set)
      - to: original.from_address (the customer)
      - cc/bcc: from DTO (optional)
      - subject: "Re: {original.subject}" (skip if already "Re:")
      - In-Reply-To: original.message_id
      - References: original.thread_id || original.message_id
   d. Send via SmtpService → smtp.gmail.com:587 (STARTTLS)
   e. Insert outbound row into `emails` table:
      - direction: 'outbound'
      - agent_name: from DTO
      - in_reply_to: original.message_id
      - thread_id: original.thread_id || original.message_id
      - Same support_email_id as original
   f. Emit WebSocket event via EmailGateway.emit_new_email()

4. Frontend receives Supabase realtime INSERT → refetches + regroups threads
   └→ Reply appears in the email's thread conversation
```

## Thread Grouping (Frontend)

Emails sharing the same `thread_id` (or `message_id` for standalone) are grouped into one `Email` card with a multi-message `thread[]` array.

**Grouping key:** `row.thread_id || row.message_id`

- Inbound email: `thread_id` is null → key = `message_id`
- Outbound reply: `thread_id` = original's `message_id` → same key

**How it works in `useEmails.ts`:**
1. `groupRowsIntoEmails()` groups all fetched rows by the key
2. Within each group, rows are sorted by `internal_date` ascending
3. First inbound row becomes the "parent" (determines from, subject, department, etc.)
4. All rows become `EmailMessage` entries in `thread[]`
5. If any row is outbound → status = 'replied'
6. Emails are sorted by most recent thread message descending

**Realtime:** On INSERT/UPDATE, the hook refetches the full page and regroups. No manual merging needed.

**selectedEmail sync:** `Emails.tsx` has a `useEffect` that keeps `selectedEmail` in sync with the `emails` array, so the open thread view updates when the realtime refetch completes.

## SMTP Configuration

- **Host:** smtp.gmail.com
- **Port:** 587 (STARTTLS)
- **Auth:** Same Gmail app password used for IMAP (stored encrypted in `support_emails.imap_password`)
- **Threading headers:** `In-Reply-To` + `References` ensure Gmail groups replies in the same thread

## CC/BCC Support

- Frontend: toggle button shows CC/BCC input fields above the reply textarea
- Comma-separated addresses parsed into arrays
- Passed through DTO → SMTP → persisted in `cc_addresses`/`bcc_addresses` columns

## DB Schema Changes

3 columns added to `emails` table:

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| direction | TEXT NOT NULL | `'inbound'` | `'inbound'` or `'outbound'` |
| agent_name | TEXT | NULL | Agent who sent the reply |
| in_reply_to | TEXT | NULL | message_id of the email being replied to |

**Index:** `idx_emails_in_reply_to` (partial, WHERE in_reply_to IS NOT NULL)

## Key Files

| File | Purpose |
|------|---------|
| `src/gmailIngestion/smtp.service.ts` | Nodemailer wrapper — SMTP transport, threading headers |
| `src/gmailIngestion/dto/sendReply.dto.ts` | DTO: body_text, body_html?, agent_name?, cc?, bcc? |
| `src/gmailIngestion/gmailIngestion.service.ts` | `send_reply()` — orchestrates fetch → decrypt → send → persist → emit |
| `src/gmailIngestion/gmailIngestion.controller.ts` | `POST /emails/:id/reply` endpoint |
| `src/gmailIngestion/gmailIngestion.module.ts` | SmtpService registered in providers |
| `src/common/crypto.ts` | `decrypt()` for SMTP password |
| `src/emailGateway/emailGateway.gateway.ts` | WebSocket broadcast (already @Global) |

## Frontend Files

| File | Purpose |
|------|---------|
| `src/hooks/useEmails.ts` | `send_reply()`, `groupRowsIntoEmails()`, realtime refetch+regroup |
| `src/pages/Emails.tsx` | Reply box UI with CC/BCC, loading spinner, optimistic update, selectedEmail sync |
| `src/integrations/supabase/types.ts` | `direction`, `agent_name`, `in_reply_to` in emails Row/Insert/Update |

## Testing

```bash
# Send reply
curl -X POST http://localhost:3002/emails/<uuid>/reply \
  -H 'Content-Type: application/json' \
  -d '{"body_text":"Test reply","agent_name":"Manish","cc":["someone@example.com"]}'

# Verify DB persistence
SELECT id, direction, agent_name, in_reply_to, thread_id
FROM emails WHERE direction = 'outbound';

# Verify thread grouping
SELECT id, direction, agent_name, message_id, thread_id
FROM emails
WHERE message_id = '<msg_id>' OR thread_id = '<msg_id>'
ORDER BY internal_date ASC;
```
