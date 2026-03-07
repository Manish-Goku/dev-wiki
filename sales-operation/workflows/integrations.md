# External Integrations

## Firebase (3 Projects)

### inventory-management-7a979 (Frontend)
- **Realtime DB**: Agent status — REST poll 5s at `/agents/{id}.json`
- **Firestore**: Order chat — `Support Orders/{orderId}/Chats` (onSnapshot)
- **Functions**: `fetchAgentsStatus()`, `getAgentStatus(id)`, `startPollingAgentStatus(id, cb, 5000)`
- **Chat**: `subscribeToOrderChats()`, `sendChatMessage()`, `sendMediaMessage()`, `markMessagesAsRead()`

### katyayani-crm-acd8c (Frontend Storage)
- **Storage**: Chat file uploads — path: `chats/{orderId}/{timestamp}_{fileName}`
- Max 10MB, types: JPEG, PNG, GIF, WebP, PDF, Word, Excel

### micro-dealer (Backend)
- **FCM**: Push notifications — `send_fcm_to_topic()`, `send_fcm_to_multiple_topics()`
  - Agronomy test notifications: email normalized to topic (`@` → `_`, `.` → `_`)
  - Data payload: `test_id`, `test_name`, `type`, `test_url`
- **Storage**: SOP file attachments — bucket: `micro-dealer.appspot.com`
  - SOP path: `sop/{sop_id}/{uuid_hex}{ext}`
  - Suggestion path: `suggestions_images/{lead_id}_{type}_{timestamp}_{uuid}.{ext}`
- Credentials: `retailers_credentials.json`

## AWS S3
- **Used for**: Principal certificate document uploads (agronomy module)
- **File types**: GST certificate, pesticide license, fertilizer license, Aadhaar front/back, PAN card
- **Path**: `agronomy/registrations/{customer_email}/{field_name}_{timestamp}.{ext}`
- **Env vars**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `S3_BUCKET_NAME`

## Slack
- **OAuth**: `GET /slack/oauth/init` → Slack authorize → callback → store token in `slack_users`
- **Messaging**: `POST /slack/send-message` → `{ channel_id, message, thread_ts?, file_data? }`
- **Scopes**: `chat:write, users:read`
- **Bot token**: `SLACK_BOT_TOKEN` env var
- **Frontend**: 15+ predefined channels, 7 message presets, @mention search (500ms debounce)

## Stockship
- **Backend proxy**: `beta.stockship.ko-tech.in` — login, raise/resolve issues, categories
- **Frontend direct**: `api.stockship.ko-tech.in` — order timeline, inventory
- **Auth**: Email/password → token in `inventory_access_token`
- **Auto-retry**: 401 → re-login → retry

## Supabase (PostgreSQL)
- **Project**: `nfudvfqwoxuqgmjjvuwj`
- **Backend**: Attendance data — `utilities/postgres_utils.py` (service role key)
- **Frontend**: `lead_profiling_trackers` table — `useProfilingTracker` hook (anon key)

## CRM Agronomy API
- **URL**: `https://beta.api.crm.ko-tech.in`
- **Auto-login**: `sahilsoni@katyayaniorganics.com` / `Sahil@123` (hardcoded in App.tsx)
- **Token**: localStorage `agronomy_access_token` with expiry
- **Used for**: Tests, question bank, answer sheets

#active #cross-project
