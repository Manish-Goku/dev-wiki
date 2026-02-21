---
workflow: Document Management
status: completed
last_worked: 2026-02-21
tags: [#completed, #active, #auto-sync]
models: [pii, contact]
modules: [pii, contacts, profile-doc-config]
---

# Document Management

PII.typed_documents is single source of truth. Contact.document_status auto-synced.

## Flow
```
API: POST/PUT/DELETE /pii/:id/typed-documents/:type
→ Mutate PII.typed_documents.{pan|adhar|pc}
→ sync_contact_licences(pii_id):
   compute_licences() → {pan: arr.length>0, adhar: ..., pc: ...}
   → Update Contact.document_status
```

## Profile Requirements
```
GET /pii/:id/document-requirements/:profile
→ ProfileDocConfigService.get_by_profile(profile)
→ Returns required/optional docs per profile type
→ 15 profile types configured (seed from static config)
```

## PiiModule Architecture
- Imports Contact schema directly (NOT ContactsModule) to avoid circular dep
- sync_contact_licences runs on every typed_documents mutation
