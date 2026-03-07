# sales-operation — Dev Wiki

Full-stack sales operations dashboard for Katyayani Organics.
**Separate project** from ko-sales-backend — Python/FastAPI backend (not NestJS), operations management (not CRM pipeline).

## Stack

| Layer | Technology | Branch |
|-------|-----------|--------|
| Backend | FastAPI (Python 3.13 local, **3.9 on EC2**), Uvicorn, pymongo | primary: `V.01`, working: `manish-master` |
| Frontend | Vite 5.4, React 18.3, TypeScript 5.8, shadcn-ui, Tailwind 3.4 | primary: `main`, working: `manish-master` |
| Database | MongoDB (`CRM-Database` — shared) | — |
| Auth | JWT HS256, bcrypt, 8hr expiry | — |
| Realtime | Firebase Realtime DB (polling), Firestore (chat) | — |
| Deploy | Backend: Render → `sales-operation-backend.ko-tech.in` | — |
| Deploy | Frontend: `agri-sales-hub.ko-tech.in` | — |

## Paths
```
~/Desktop/sales-operation/
├── sales-operation-backend-1/     # FastAPI backend
└── sales-operations/              # React frontend
```

## Commands
```bash
# Backend
cd ~/Desktop/sales-operation/sales-operation-backend-1
pip install -r requirements.txt
uvicorn main:app --reload --port 8000

# Frontend
cd ~/Desktop/sales-operation/sales-operations
npm run dev    # port 5173, proxy /api→:8000
npm run build  # vite build → dist/
```

## Backend Modules
```
main.py                          # Entry, lifespan (MongoDB, Firebase, AssignmentSystem, DBListener)
auth/                            # JWT auth, signup, role CRUD, module access permissions
dashboard/                       # Order KPIs, pending, NDR, escalated, failed, tracking, agent-details (with call audits + skill scores)
leads_operation/                 # Lead analytics, batches, clusters, workflows, assignment logs
leads_management/                # Lead request approval flow (pending→approved→given→fulfilled)
Inventory/                       # SKU inventory listing (warehouse OID: 674872800a3c153036bc00f0)
agronomy/                        # Call audits (dynamic skills, $lookup audit status, server-side search, stats), tests, question bank, suggestions, skill matrix (precomputed)
sop/                             # SOP CRUD with Firebase Storage attachments
Reports_and_Analytics/           # Manager/agent revenue, performance, attendance (Supabase)
coupons/                         # Coupon CRUD, rule engine (nested conditions), eligibility, apply, usage logs
stockship_proxy/                 # Proxy to beta.stockship.ko-tech.in (avoids CORS)
utilities/db_listener.py         # MongoDB change stream → auto lead assignment
utilities/assignment_logic.py    # Workflow engine, round-robin tracker
utilities/slack/                 # Slack OAuth + bot messaging
utilities/fcm_utils.py           # Firebase Cloud Messaging (push notifications)
utilities/postgres_utils.py      # Supabase PostgreSQL (attendance)
workers/lifecycle.py             # Assignment system lifecycle (start/stop/health)
```

## Frontend (54 pages, 50+ hooks)
| Section | Pages |
|---------|-------|
| Dashboard | KPIs, revenue, recent orders, active agents |
| Orders | AllOrders, SupportPending, PreDispatch, PreDelivery, NDR, RTO, Failed |
| Inventory | OutOfStock |
| Pricing | SpecialPricing, Coupons |
| Leads | LeadRequests, LeadAssignment |
| Lead Master (BA) | Inflow, Analysis, BatchAssign, Performance, Workflows, Quality |
| Agronomy | CallAudits, Tests, TestView, TakeTest, PC Certificates, Suggestions |
| Training | NewJoining, SkillMatrix, SOPs |
| Reports | FloorManager, SalesHead, AgentActivity |
| Admin | UserManagement, RoleManagement |
| Other | CoinMaster, HappinessIndex, ProfileTracker, MRVTracker |

## Auth & Roles
- **Roles**: `sales_admin, sales_head, floor_manager, support_manager, junior_support, senior_agronomy, junior_agronomy, ba_team`
- Default signup role: `retention` (can't login with it)
- Module access: `permissions_matrix` collection → sidebar filtering
- Frontend localStorage: `backend_access_token, backend_user_email, backend_user_roles, user_password (base64)`

## External Integrations
| Service | Used For | Access |
|---------|---------|--------|
| Firebase Realtime DB | Agent status (5s poll) | Frontend direct |
| Firebase Firestore | Order support chat | Frontend direct (onSnapshot) |
| Firebase Storage | Chat uploads, SOP files, suggestion photos | Frontend + Backend |
| Firebase FCM | Push notifications (tests, escalations) | Backend only |
| AWS S3 | Principal certificate doc uploads | Backend only |
| Slack API | OAuth + messaging | Backend proxy |
| Stockship API | Inventory + order tracking | Both (proxy + direct) |
| Supabase PostgreSQL | Attendance, profiling tracker | Both |
| CRM Agronomy API | Tests, question bank | Frontend (auto-login) |

## Models Index
| Model | Collection | Doc |
|-------|-----------|-----|
| User | `users` | [users.md](models/users.md) |
| Agent | `agents` (shared) | [agents.md](models/agents.md) |
| Order | `orders` (shared) | [orders.md](models/orders.md) |
| Lead | `leads` (shared) | [leads.md](models/leads.md) |
| Call | `calls` (shared) | [calls.md](models/calls.md) |
| Batch | `batches` | [batches.md](models/batches.md) |
| Workflow | `workflows` | [workflows.md](models/workflows.md) |
| Cluster | `clusters` | [clusters.md](models/clusters.md) |
| SOP | `sops` | [sops.md](models/sops.md) |
| Call Audit | `call_audit` | [call_audit.md](models/call_audit.md) |
| Agronomy Test | `agronomy_tests, agronomy_questions, agronomy_answer_sheets` | [agronomy_tests.md](models/agronomy_tests.md) |
| Agronomy Suggestion | `agronomy_suggestions` | [agronomy_suggestions.md](models/agronomy_suggestions.md) |
| Principal Certificate | `principal_certificates` | [principal_certificates.md](models/principal_certificates.md) |
| Permissions | `roles, modules, permissions_matrix` | [permissions.md](models/permissions.md) |
| Coupon | `sales_coupons, sales_coupon_usage` | [sales_coupons.md](models/sales_coupons.md) |

## Workflows Index
| Workflow | Doc |
|----------|-----|
| Auth & Module Access | [auth-module-access.md](workflows/auth-module-access.md) |
| Lead Auto-Assignment | [lead-auto-assignment.md](workflows/lead-auto-assignment.md) |
| Order Pipeline | [order-pipeline.md](workflows/order-pipeline.md) |
| Frontend-Backend Mapping | [frontend-backend-mapping.md](workflows/frontend-backend-mapping.md) |
| Integrations | [integrations.md](workflows/integrations.md) |
| Agronomy Module | [agronomy-module.md](workflows/agronomy-module.md) |
| Training Module | [training-module.md](workflows/training-module.md) |
| Coupon System | [coupon-system.md](workflows/coupon-system.md) |

## Key Constants
| Constant | Value | Location |
|----------|-------|----------|
| SLA duration | 4 working hours (10AM-7PM) | dashboard/route.py, lib/slaCalculator.ts |
| Unassigned agents | A0-1126 to A0-1130 | utilities/db_listener.py |
| Lead debounce | 10 seconds | utilities/db_listener.py |
| Token expiry | 480 min (8 hours) | auth/utils.py |
| Cache TTL | 5 min | frontend lib/cache.ts |
| Agent poll | 5 seconds | frontend lib/firebase.ts |
| Max file upload | 10 MB | frontend lib/firebaseStorage.ts |

## DB Field Gotchas
- `State` (capital S) in leads — NOT `state`
- `assigned_Date` (capital D) in leads
- `desposition` TYPO in calls — NOT `disposition`
- `Agent_Category` (capitals) in agents
- `total_amt` in orders — NOT `total_amount`
- `date` field for range filtering in orders — NOT `order_date`
- `slackId` vs `slack_id` — both exist in agents

#active #cross-project
