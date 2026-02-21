# ko-sales-backend

NestJS 10 + TypeScript sales CRM backend for Katyayani Organics.

**DB:** MongoDB (`ko-sales-db`) | **Port:** 3000 | **Prefix:** `/api/v1/`

## Models

| Model | Collection | ID Pattern | Doc |
|-------|-----------|------------|-----|
| [Lead](models/lead.md) | leads | K0-1XXXXX | Schema + indexes |
| [Contact](models/contact.md) | contacts | C-N | Schema + indexes |
| [Customer](models/customer.md) | customers | CT-N | Schema + indexes |
| [Deal](models/deal.md) | deals | D-N | Schema + embedded quotation |
| [PII](models/pii.md) | piis | PII-N | Privacy layer |
| [Agent](models/agent.md) | agents | A0-XXXX | Auth + assignment |
| [Pipeline](models/pipeline.md) | lead/contact/deal_pipelines | LP/CP/DP-N | Stage tracking |
| [PipelineWorkflow](models/pipeline-workflow.md) | pipelines_workflow | PW-N | Stage config |
| [Task](models/task.md) | tasks | T-N | SLA + automation |
| [TaskWorkflow](models/task-workflow.md) | task_workflows | TW-N | Step config |
| [Cohort](models/cohort.md) | cohorts | COH-N | Rule-based segmentation |
| [Signal](models/signal.md) | signals | SIG-XXXXX | Inbound signals |
| [Note](models/note.md) | notes | ObjectId | Standalone notes |
| [Call](models/call.md) | calls | CALL-N | Call logs |
| [DraftOrder](models/draft-order.md) | draft_orders | DFT-N | Order drafts |
| [CohortAssignmentJob](models/cohort-assignment-job.md) | cohort_assignment_jobs | CAJ-N | Async assignment tracking |

## Workflows

| Flow | Doc | Tags |
|------|-----|------|
| [Lead Lifecycle](workflows/lead-lifecycle.md) | Create → Update → Assign → Convert | `#completed` `#active` |
| [Pipeline Transitions](workflows/pipeline-transitions.md) | Stage validation + task triggers | `#completed` `#active` |
| [Task Automation](workflows/task-automation.md) | Pipeline → Task → SLA → Auto-advance | `#completed` `#active` |
| [Deal & Quotation](workflows/deal-quotation.md) | Deal CRUD + quotation lifecycle | `#completed` `#active` |
| [Cohort System](workflows/cohort-system.md) | Rule engine + evaluation + bulk assignment | `#active` `#event-driven` `#bullmq` |
| [Customer Sync](workflows/customer-sync.md) | Inventory-Management → Customer | `#completed` `#webhook` `#cross-project` |
| [Document Management](workflows/document-management.md) | PII typed_documents → Contact licences | `#completed` `#active` |
| [SLA Engine](workflows/sla-engine.md) | Business hours + breach detection | `#completed` `#active` |
| [Assignment Cascade](workflows/assignment-cascade.md) | Lead assign → pipeline + contact cascade | `#completed` `#active` |

## Last Worked

| Date | What |
|------|------|
| 2026-02-21 | Customer module + Cohort system (hybrid rule engine, event-driven evaluation) |
| 2026-02-21 | Cohort bulk assignment: BullMQ + Redis, capacity-weighted distribution, async job tracking |
