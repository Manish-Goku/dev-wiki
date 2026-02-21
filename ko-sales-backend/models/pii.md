---
model: PII
collection: piis
id_pattern: PII-N (e.g. PII-1)
status: active
last_updated: 2026-02-21
tags: [core, privacy, documents]
depends_on: []
referenced_by: [lead, contact, customer, deal, signal, note, call, cohort]
---

# PII (Personally Identifiable Information)

Privacy layer — single source of truth for sensitive data. All entities reference PII by `pii_id` string (NOT ObjectId, no .populate()).

## Fields

| Field | Type | Notes |
|-------|------|-------|
| pii_id | String | Unique, PII-N |
| phone_number | String[] | Unique index |
| email | String | |
| country_code | String | |
| addresses | Array | `[{label, line1, line2, city, state, pincode, country}]` |
| gst_numbers | Array | `[{gst_number, business_name, status, registration_date}]` |
| typed_documents | Object | `{pan: [{id, image}], adhar: [{id, image}], pc: [{id, image}]}` |
| extension | String | |

## Key Rules

- One PII per phone number (unique index)
- typed_documents mutations auto-sync Contact.document_status via `sync_contact_licences()`
- PiiModule imports Contact schema directly (avoids circular dep)
- Manual joins required (string ref, not ObjectId)
