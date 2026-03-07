# skill_matrix

**Collection**: `skill_matrix` (in CRM-Database)
**Used by**: `agronomy/` module (skill matrix endpoint + refresh on audit create/update)

## Schema
```javascript
{
  agent_email: String,       // unique, indexed — agent's email (from calls.agent_id)
  agent_name: String,        // "Priti Sasmal" (from agents.firstname + lastname)
  agent_category: String,    // "Retention", "Retailer-KYC", etc. (from agents.Agent_Category), indexed
  overall_score: Number,     // avg of all skill averages (e.g., 4.26)
  audit_count: Number,       // total audits for this agent, indexed, used for sorting
  skills: [                  // precomputed skill breakdown
    {
      skill: String,         // display name, e.g., "Call Opening"
      code: String,          // skill code, e.g., "call_opening"
      score: Number          // avg score across all audits (e.g., 7.5)
    }
  ],
  updated_at: DateTime       // last refresh timestamp
}
```

## Indexes
| Field | Type | Name |
|-------|------|------|
| `agent_email` | unique | `idx_agent_email` |
| `agent_category` | regular | `idx_agent_category` |
| `audit_count` | regular | `idx_audit_count` |

## How It Works

### Precomputed (not live aggregation)
This collection stores **precomputed** skill averages per agent. It is NOT computed on-the-fly.

### Write Path: `refresh_agent_skill_matrix(db, agent_email)`
Called automatically from:
- `create_call_audit()` — after inserting a new audit
- `update_call_audit()` — after updating an existing audit

Logic:
1. Aggregate all `call_audit` records for the given `agent_email`
2. Compute `$avg` for each active skill from `skills_library`
3. Look up agent name/category from `agents` collection
4. Compute `overall_score` as percentage: `(sum_of_avg_scores / sum_of_max_scores) * 100`
   - e.g., if avg scores sum to 120 and max_score values sum to 170 → `overall_score = 70.6`
   - This is more accurate than naive "average of averages" which doesn't account for different max_score values
5. Upsert into `skill_matrix` (insert if new, update if exists)
6. If no audits remain for an agent, delete their `skill_matrix` entry

### Read Path: `get_skill_matrix(db, email, department, page, limit)`
Simple `find()` query:
```python
results = skill_matrix_col.find(query, {"_id": 0}).sort("audit_count", -1).skip(...).limit(...)
```
Supports: email filter, department filter, pagination (page/limit).

### Performance
- **Read**: ~0.3s (simple find with indexes)
- **Write**: ~0.1s per agent refresh (small aggregation on single agent's audits)
- **Previous approach**: 75s (aggregation pipeline with `$facet` on shared MongoDB — IOPS throttled)

## Initial Population
Script: `populate_skill_matrix.py` in backend repo root. Run once to seed from existing `call_audit` data.

## Relationship to call_audit
`call_audit` stores denormalized `agent_email` and `agent_category` fields (set at audit creation time from `calls` + `agents`). The refresh function uses `agent_email` to match and aggregate audits.

## API Endpoint
| Method | Path | Params | Purpose |
|--------|------|--------|---------|
| GET | `/agronomy/skill-matrix` | `email`, `department`, `page` (default 1), `limit` (default 25) | Read precomputed skill matrix |

## Frontend
- **Hook**: `useSkillMatrix(page, limit, department)` → GET `/api/agronomy/skill-matrix`
- **Page**: SkillMatrix.tsx (`/training/skills`) — Agent Skills tab
- **Pagination**: Server-side, Prev/Next buttons
- **Department filter**: Server-side, options from `/agronomy/agent-categories`

#active
