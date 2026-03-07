# agronomy_tests

**Collection**: `agronomy_tests` (in CRM-Database)
**Used by**: `agronomy/` module

## Schema
```javascript
{
  _id: ObjectId,
  test_id: String,                          // e.g. "TEST-1001"
  test_name: String,
  total_marks: Number,
  agent_category: String,                   // optional department filter
  specific_agents: [String],                // katyayani_ids assigned
  questions: [ObjectId],                    // refs agronomy_questions._id
  created_by: String,                       // email
  live_on: DateTime,                        // when test becomes available
  tags: [String],
  state: String,                            // optional
  month: String,
  given_by: [String],                       // agents who completed
  pending_agents: [String],                 // agents who haven't (initially = specific_agents)
  createdAt: DateTime,
  updatedAt: DateTime
}
```

## Related Collections

### agronomy_questions
```javascript
{
  _id: ObjectId,
  question_type: String,                    // "mcq", "true_false"
  question: String,
  image_urls: [String],
  mcq_options: [String],                    // ["Option A", "Option B", ...]
  answer: String,                           // must be in mcq_options
  difficulty: String,                       // "easy", "medium", "hard"
  topic: String,
  marks: Number,                            // default 1
  created_by: String
}
```

### agronomy_answer_sheets
```javascript
{
  _id: ObjectId,
  katyayani_id: String,                     // agent who took test
  test_id: String,
  answers: [{
    question_id: String,
    submitted_answer: String,
    marks_obtained: Number                  // 0 or full marks
  }],
  total_marks_obtained: Number,
  out_of: Number,
  is_checked: Boolean,                      // true for auto-evaluated MCQ
  evaluated_by: String,                     // "SYSTEM" for auto
  createdAt: DateTime,
  updatedAt: DateTime
}
```

#active
