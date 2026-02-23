# Gmail Ingestion Flow

`#wip` `#webhook` `#cron` `#event-driven`

## Overview

Real-time email ingestion from Gmail using push notifications via Pub/Sub.

## Flow

```
1. Admin adds support email via POST /support-emails
   └→ Record created in support_emails table
   └→ Gmail API watch() called → Pub/Sub topic registered
   └→ watch_expiration + watch_history_id stored

2. Customer sends email to support@katyayaniorganics.com
   └→ Gmail detects new message in INBOX
   └→ Publishes notification to Pub/Sub topic

3. Pub/Sub pushes to POST /webhooks/gmail
   └→ Decode base64 payload → { emailAddress, historyId }
   └→ Lookup support_email by email_address
   └→ Fetch new messages since stored history_id via Gmail history.list()

4. For each new message:
   └→ Dedup check (gmail_message_id exists?)
   └→ Fetch full message via Gmail messages.get()
   └→ Parse headers, body (text/html), attachments
   └→ Gemini AI: summarize + classify team
   └→ Insert into emails table (with summary + suggested_team)
   └→ WebSocket emit: new_email → inbox room + broadcast

5. Update watch_history_id to latest
```

## Cron: Watch Renewal

- **Schedule:** Every 6 hours (`0 */6 * * *`)
- **Logic:** Check all active support_emails. If watch_expiration < 24h from now, call watch() again
- **File:** `src/gmailIngestion/gmailCron.service.ts`

## Auth: Domain-Wide Delegation

- Service account impersonates each monitored user via JWT
- Scope: `gmail.readonly`
- No per-user OAuth consent needed (Workspace admin authorizes once)

## Key Files

| File | Purpose |
|------|---------|
| `gmail.service.ts` | Gmail API wrapper (auth, watch, fetch, parse) |
| `gmailIngestion.service.ts` | Business logic (CRUD, webhook processing) |
| `emailAi.service.ts` | Gemini summarization + team classification |
| `gmailWebhook.controller.ts` | POST /webhooks/gmail endpoint |
| `gmailCron.service.ts` | Watch renewal cron |
| `emailGateway.gateway.ts` | WebSocket push to frontend |
