---
workflow: Pipeline Transitions
status: completed
last_worked: 2026-02-21
tags: [#completed, #active, #task-trigger]
models: [pipeline, pipeline-workflow, task, task-workflow]
modules: [pipelines, tasks]
---

# Pipeline Transitions

`PUT /pipelines/:pipeline_id/transition` → target_stage

## Flow

```
Find workflow config (by pipeline.workflow_id)
→ Validate: target stage exists? forward-only? required fields met?
→ Close current stage (set exited_at)
→ Open new stage (push to stage_history, set current_stage)
→ Cancel old stage pending tasks
→ Trigger tasks for new stage (if taskflow_id exists)
```

## Validation

- Target stage must exist in workflow
- `target.index > current.index` (forward-only)
- Required fields checked on entity (supports dot-path like `call_count.answered`)
- Auto-advance bypasses required_fields check

## Task Integration

- Stage has `taskflow_id` → find TaskWorkflow → create step 1 task
- Duplicate prevention: skip if step already exists for pipeline+workflow combo
- Task completion → next step → all done → auto-advance pipeline to next stage
