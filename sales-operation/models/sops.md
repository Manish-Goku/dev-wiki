# sops

**Collection**: `sops` (in CRM-Database)
**Used by**: `sop/` module

## Schema
```javascript
{
  _id: ObjectId,
  sop_id: String,                      // "SOP-{uuid4_hex[:8].upper()}"
  title: String (1-200),
  description: String (max 1000),
  content: String (min 1),
  category: String (max 100),
  updated_by: String,
  version: Number,                     // incremented on update
  tags: [String],
  attachment: {
    filename: String,
    url: String,                       // Firebase Storage URL
    size: Number,
    content_type: String,
    uploaded_at: DateTime
  },
  created_at: DateTime,
  updated_at: DateTime
}
```

## API
- `POST /sop/sops` — create (with optional file attachment via Firebase Storage)
- `GET /sop/sops?page=&limit=&search=&tags=&category=` — list (search: regex on title/description/content)
- `GET /sop/sops/{sop_id}` — single
- `PUT /sop/sops/{sop_id}` — update (increments version)
- `DELETE /sop/sops/{sop_id}` — delete

## Storage
Firebase bucket: `micro-dealer.appspot.com`

#active #auth-required
