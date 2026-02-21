---
model: Agent
collection: agents
id_pattern: A0-XXXX (e.g. A0-1001)
status: active
last_updated: 2026-02-21
tags: [core, auth, assignment]
depends_on: []
referenced_by: [lead, contact, deal, task]
---

# Agent

Sales agents and admin users.

## Fields

| Field | Type | Notes |
|-------|------|-------|
| agentId | String | Unique, A0-XXXX |
| firstname / lastname | String | |
| email | String | Unique |
| katyayani_id | String | External ID |
| user_role | String | USR-1000 to USR-1005 |
| Agent_Category | String | Acquisition, Retention, Support, Company B2B |
| AssignedState | String[] | States for round-robin |
| status | String | |
| slack_id | String | |
| is_active | Boolean | |
| password | String | select: false |
| refreshToken | String | |

## Role Codes

| Code | Role | Admin? |
|------|------|--------|
| USR-1000 | Super Admin (max 4) | Yes |
| USR-1001 | Admin | Yes |
| USR-1002 | Default (sales agent) | No |
| USR-1003 | Team Manager | No |
| USR-1004 | Floor Manager | Yes |
| USR-1005 | Agro | Yes |
