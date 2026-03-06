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
| Associate | associates | KAS-XXXXXX | Partner onboarding + compliance |
| FavouriteProduct | favourite_products | — | Per-associate product wishlist |
| Address | address_v2 | — | Separated address records |
| AssociateEvent | associate_events | — | Event stream (immutable log) |
| AssociateFeature | associate_features | — | APS scores + metrics |
| OnboardingConfig | onboardingconfigs | — | Dynamic form schema |
| OnboardingProgress | partner_onboarding_progress | — | Step-by-step user journey |
| Agreement | agreements | — | Partner T&Cs + contracts |
| Product | products | SKU (unique) | Product catalog |
| Scheme | schemes | — | Promotional campaigns |
| SchemeUsage | scheme_usage | — | Campaign usage tracking |
| Tier | tiers | tierCode | Bronze/Silver/Gold/Diamond |
| WalletTransaction | wallet_transactions | — | Coin ledger |
| Ticket | tickets | — | Support tickets |
| [BulkV2](models/bulk-v2.md) | bulk_v2 | SKU (unique) | B2B product catalog + CRUD module |

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
| [Associate Catalog](workflows/associate-catalog.md) | Visibility + pricing + images + packaging | `#active` `#rule-engine` |
| [Bulk Catalog CRUD](workflows/bulk-catalog-crud.md) | Direct bulk product CRUD (7 endpoints) | `#active` `#crud` |

## Associate & B2B Commerce Modules (merged from mayank-dev, 2026-02-28)

13 new modules merged from `mayank-dev` branch — partner onboarding, APS scoring, schemes, wallet, tiers, payments, tickets, products, cart.

### New Module Route Map

| Route Prefix | Module | Purpose |
|-------------|--------|---------|
| /associates | associates | Partner CRUD, search, enrichment, favourite products |
| /admin/associates | associates | Admin management, stats, approval |
| /profile/get | associates | Mobile profile endpoint |
| /associate-events | associate-events | Event ingestion (single + batch) |
| /associate-features | associate-features | APS-sorted priority list |
| /agreements | agreements | Partner agreement CRUD |
| /onboarding-config/* | onboarding-config | Dynamic form schema (public + admin) |
| /onboarding-progress/:userId | onboarding-progress | Progress tracking + resume |
| /products | products | Catalog with filters, featured, new |
| /cart/add | cart | Add to cart |
| /schemes/active | schemes | Filtered active schemes for user |
| /schemes/evaluate | schemes | Evaluate benefits for order |
| /admin/schemes/* | schemes | Admin CRUD, pause, activate, analytics |
| /tickets | tickets | Support ticket CRUD + comments |
| /tiers | tiers | Tier listing + user progress |
| /wallet | wallet | Balance + recent transactions |
| /bulks | bulks | BulkV2 CRUD: create, list, get, update, delete, categories |

### Favourite Products
- **Collection**: `favourite_products` — one document per associate+SKU pair
- **Schema**: `associate_id` (user_id string), `sku`, `added_at`
- **Index**: `{ associate_id: 1, sku: 1 }` unique compound
- **Endpoints**: `POST /associates/:id/favourites` (add), `DELETE /associates/:id/favourites/:sku` (remove), `GET /associates/:id/favourites` (list, newest first)

### Key Architectural Patterns
- **Event Sourcing**: Associate events → immutable log → APS scoring engine
- **APS (Automated Priority Score)**: Weights — intent 40%, engagement 25%, timing 20%, cost 15%. Actions: ≥0.85 "Call now", ≥0.65 "Same-day follow-up", ≥0.45 "WhatsApp", else "Deprioritize"
- **Rule Engine (Schemes)**: targeting matcher → condition evaluator (AND/OR) → action executor. Supports wallet_bonus, order_discount, product_discount, coin_loading
- **Aggregate Service**: AssociateAggregateService merges Associate+PII+Address with lazy loading
- **Multilingual Config**: OnboardingConfig supports en/hi labels, conditional field visibility
- **Juspay Payment**: Headless gateway for wallet top-ups (K_<timestamp> order IDs)
- **Auth Extension**: OTP-based associate login + Supabase JWT strategy added

### Existing Module Changes
- **Auth**: OTP service, associate auth endpoints, Supabase JWT strategy
- **Agents**: Hierarchy support, agent_category sync, team filtering
- **Dashboard**: Restricted to 4 categories, contact pipeline funnel
- **Contacts**: Stage filtering, segment-aware view
- **Deals**: Quotation management, associate-based assignment
- **Leads**: Agent/date-range filtering
- **Pipelines**: Workflow versioning, publish/activate
- **Signals**: Query optimization (13s → 0.18s with post-filter)
- **PII**: Phone normalization, separated storage, duplicate ID fix

## Customer Success Module (merged 2026-02-28)

All 26 CS sub-modules merged under `src/modules/customer-success/` as an isolated `CustomerSuccessModule`.

- **169 files**, 53 directories — zero overlap with existing ko-sales modules
- **All routes**: `/api/v1/cs/*` (prefixed to avoid collisions)
- **All Swagger tags**: `cs/*` (31 tags in `main.ts`)
- **Auth**: `@Public()` on all CS controllers (bypasses JWT guard)
- **DB**: Separate Supabase project via `CsSupabaseService` (`CS_SUPABASE_URL` env var)
- **Branches**: Merged into `manish-master-v2` + `praveen` (zero conflicts)
- **Full docs**: See `~/Desktop/dev-wiki/customer-success-backend/README.md` for 160+ endpoint details

### CS Route Map

| Route Prefix | Module | Purpose |
|-------------|--------|---------|
| cs/support-emails | gmailIngestion | Manage monitored email addresses |
| cs/emails | gmailIngestion | View ingested emails |
| cs/conversations | chatIngestion | WhatsApp conversations |
| cs/messages | chatIngestion | Chat messages |
| cs/webhooks | chatIngestion | Interakt/Netcore webhooks |
| cs/webhooks/ivr | ivrWebhooks | IVR call data |
| cs/agents | agents | CS agent management |
| cs/roles | roles | Role hierarchy |
| cs/tickets | tickets | Ticket CRUD |
| cs/ticket-categories | tickets | Categories |
| cs/ticket-templates | tickets | Response templates |
| cs/ticket-routing-rules | tickets | Auto-routing |
| cs/notifications | notifications | Notifications + approvals |
| cs/admin/dashboard | adminDashboard | Analytics |
| cs/chat-templates | chatTemplates | Canned responses |
| cs/query-assignments | queryAssignments | Agent dispatch |
| cs/sla-config | slaConfig | SLA settings |
| cs/refunds | refunds | Refund management |
| cs/audits | audits | Audit logs |
| cs/survey-templates | surveys | Survey templates |
| cs/survey-campaigns | surveys | Survey campaigns |
| cs/survey-calls | surveys | Survey call tracking |
| cs/workflows | workflows | Workflow automation |
| cs/video-call-leads | videoCallLeads | Video call leads |
| cs/call-categories | callCategories | Call categorization |
| cs/youtube-leads | youtubeLeads | YouTube leads |
| cs/customer-feedback | customerFeedback | Customer feedback |
| cs/customer-timeline | customerTimeline | Activity timeline |
| cs/agent-feedback | agentFeedback | Agent performance |
| cs/return-rca | returnRca | Return root cause |
| cs/agri-consultancy | agriConsultancy | Agri consultancy |

## Last Worked

| Date | What |
|------|------|
| 2026-03-06 | BulkV2 CRUD module (7 endpoints: create, list, get, update, delete, categories, sub-categories). Fixed app.module.ts duplicate imports, agents.service.ts duplicate method, installed @aws-sdk/client-s3 |
| 2026-03-05 | Cohort system tested in depth (28/30 passing). Fixed: operator validation in rule-compiler, BSON 16MB overflow cap at 500k members |
| 2026-02-28 | Quotation LineItem schema extended: added requested_weight, packaging_size, uom, packaging_type for associate app bulk quotations |
| 2026-02-28 | mayank-dev merged into manish-master-v2 (13 new modules: associates, schemes, wallet, tiers, tickets, payment, cart, products, onboarding, APS, agreements, events, features) |
| 2026-02-28 | Customer Success module merged into ko-sales-backend (manish-master-v2 + praveen) |
| 2026-02-21 | Customer module + Cohort system (hybrid rule engine, event-driven evaluation) |
| 2026-02-21 | Cohort bulk assignment: BullMQ + Redis, capacity-weighted distribution, async job tracking |
