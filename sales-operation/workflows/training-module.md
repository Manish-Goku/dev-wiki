# Training Module

Covers SOP management, new joining induction, and skill-based training programs.

## Backend Structure

```
sop/
  __init__.py
  route.py           # 5 CRUD endpoints
  queries.py          # MongoDB queries for sops collection
  request_model.py    # CreateSOPRequest, UpdateSOPRequest
  response_model.py   # SOPItem, AttachmentInfo, CRUD responses
  storage_utils.py    # Firebase Storage upload/delete
```

**Auth**: All endpoints require `require_operation_sales_role`.

---

## 1. SOP Management

### Endpoints
| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | `/sop/sops` | operation_sales_role | Create SOP |
| GET | `/sop/sops` | operation_sales_role | List SOPs (paginated, searchable) |
| GET | `/sop/sops/{sop_id}` | operation_sales_role | Get single SOP |
| PUT | `/sop/sops/{sop_id}` | operation_sales_role | Update SOP (increments version) |
| DELETE | `/sop/sops/{sop_id}` | operation_sales_role | Delete SOP + attachment |

### Request Models
**CreateSOPRequest**:
- `title` (str, 1-200 chars, required)
- `description` (str, max 1000 chars, optional)
- `content` (str, min 1 char, required)
- `category` (str, max 100 chars, optional) - values: "process", "training", "portal"
- `tags` (List[str], optional)

**UpdateSOPRequest**: Same fields, all optional.

### Response Model: SOPItem
```
sop_id, title, description, content, category, updated_by,
version (int), tags[], attachment (AttachmentInfo), created_at, updated_at
```

### Collection: `sops`
```javascript
{
  sop_id: "SOP-{uuid4_hex[:8]}",
  title, description, content, category,
  updated_by: String (user email),
  version: Number (starts 1, increments on update),
  tags: [String],
  attachment: { filename, url, size, content_type, storage_path, uploaded_at },
  created_at, updated_at
}
```

### Query Features
- **Search**: regex on title, description, content (case-insensitive)
- **Tags filter**: `$in` operator on comma-separated tags
- **Category filter**: regex (case-insensitive)
- **Sort**: `updated_at` DESC
- **Pagination**: skip = (page-1) * limit

### File Storage
- Firebase bucket: `micro-dealer.appspot.com`
- Path: `sop/{sop_id}/{uuid4_hex}{extension}`
- `upload_sop_attachment(file, sop_id)` -> public URL
- `delete_sop_attachment(storage_path)` -> cleanup on SOP delete

### Frontend Page: SOPs.tsx - `/training/sops`
- **Hook**: `useSOPs()` -> `fetchSOPs(page, limit)`, `createSOP(data)`, `updateSOP(id, data)`
- **Access control**: Requires `reports_analytics` module + `training_sops` submodule
- **Stats**: Total SOPs, Process SOPs, Training SOPs, Portal Guides
- **Search**: Text search + tag filtering
- **Category icons**: process=FileText, training=BookOpen, portal=Video
- **Dialogs**: Create, Detail (read-only), Edit
- **PDF download**: jsPDF with title, metadata, content, footer
- **Table columns**: Title, Tags (max 3 shown), Category, Version, Updated By, Last Updated, Actions

### Missing Features (UI exists but no backend)
- Delete button not exposed in UI (backend endpoint exists)
- File upload UI not implemented (backend supports it)
- View History button exists but no implementation
- Version history tracking not built

---

## 2. New Joining Induction

### Frontend Page: NewJoining.tsx - `/training/induction`

**Status**: UI only, no backend API integration. Uses empty mock data.

### 7 Induction Stages
```
waiting -> basic_training -> crm_training -> video_content
  -> skill_assessment -> test -> ready
```

### Stats
- Total Joinings, Waiting, In Training, Ready, SLA Breach, Avg Progress

### Table Columns
- New Joinee (avatar, name, age/gender)
- Role & Department
- Joining Date & Experience
- Videos watched (e.g., 5/20)
- SLA Status (days in stage or breach alert)
- Current Stage (colored badge)
- Progress bar
- Trainer
- Actions: View Profile, Update Status, Assign Test, Mark Ready

### Detail Dialog
- Profile header with progress & SLA
- 7-stage training timeline (completed/current/pending dots)
- Video progress card
- Additional info grid

### Filters
- Stage (all 7), Department (Acquisition/Retention), SLA Status

---

## 3. Training Programs

### Frontend: Part of SkillMatrix.tsx - `/training/skills` (Trainings tab)

**Status**: UI only, no backend persistence.

### Create Training Dialog
- Agent selection (checkbox list)
- Skill selection, Current Level, Target Level (A/B/C)
- Start Date, Closure Date
- Trainer Name, Description

### Training Table Columns
- Agent, Skill, Level Change (A->B), Timeline, Trainer, Status, Actions

---

## Frontend Routes
| Route | Page | Status |
|-------|------|--------|
| `/training/sops` | SOPs | Fully functional |
| `/training/induction` | NewJoining | UI only (no backend) |
| `/training/skills` | SkillMatrix | Functional (skill matrix with server-side dept filter + pagination, agent detail dialog with audits, training programs UI only) |

## Navigation
- "Agronomy & Training" group: New Joining, Skill Matrix
- "Reports & Analytics" group: Training SOPs

## Hooks
| Hook | Endpoints | Status |
|------|-----------|--------|
| `useSOPs` | GET/POST/PUT `/sop/sops` | Functional |
| `useSkillMatrix` | GET `/api/agronomy/skill-matrix` | Functional |

#active #auth-required
