# skills_library

**Collection**: `skills_library` (in CRM-Database)
**Used by**: `agronomy/` module

## Schema
```javascript
{
  _id: ObjectId,
  code: String,           // unique key, e.g., "call_opening", "testing_skill"
  name: String,           // display name, e.g., "Call Opening"
  category: String,       // e.g., "Communication", "Sales"
  description: String,    // optional description
  importance: String,     // "high", "medium", "low"
  max_score: Number,      // max score for this skill (typically 10)
  is_system: Boolean,     // true = built-in skill, cannot be deleted
  is_active: Boolean,     // only active skills appear in audit forms
  created_at: DateTime,
  updated_at: DateTime
}
```

## Endpoints
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/agronomy/skills` | List all skills |
| POST | `/agronomy/skills` | Create new skill |
| PUT | `/agronomy/skills/{skill_id}` | Update skill |
| DELETE | `/agronomy/skills/{skill_id}` | Delete skill (system skills cannot be deleted) |
| GET | `/agronomy/call-audits/parameters` | Returns only active skills for audit form rendering |

## How Skills Drive the System

1. **Audit Form**: Frontend fetches active skills via `/call-audits/parameters`, renders one input per skill
2. **Audit Submission**: Scores sent as `{code: score}` dict, stored as top-level fields in `call_audit`
3. **Total/Percentage**: `total_score` = sum of scores, `max_total_score` = sum of active skills' max_score, `percentage_score` = (total/max)*100
4. **Skill Matrix**: Aggregation pipeline dynamically builds `$avg` for each skill code
5. **View Details**: Frontend maps audit fields to skill names using parameters endpoint

## Default System Skills (14)
call_opening, call_closing, product_clarity, farmer_engagement, solution_relevance, objection_handling, urgency_creation, buying_signal_response, product_recommendation, follow_up_plan, overall_communication, information_gathering, call_confidence, farmer_satisfaction

## Frontend
- Managed in **Skills Library** tab on SkillMatrix page (`/training/skills`)
- CRUD operations via the 4 endpoints above
- Skills appear dynamically in Complete Audit and View Details dialogs

#active
