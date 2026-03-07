# Frontend ↔ Backend Mapping

## By Page

### Dashboard (`/`)
- `useDashboardSummary` → `GET /dashboard/orders-summary?start_date=&end_date=&limit=&page=`
- `useActiveAgents` → Firebase Realtime DB `/agents.json` (5s poll)

### Orders
- All Orders: `useRetailerOrders` → `GET /dashboard/others-retailers-orders?page=&limit=&start_date=&end_date=&order_id=&agent_category=&order_status=&payment_mode=`
- Order Detail: `useOrderDetails` → `GET /dashboard/order-details?order_id=`
- Order Timeline: `useStockshipOrderTimeline` → Stockship `GET /orders?exact_order_id=`
- Order Chat: `useChat` → Firestore `Support Orders/{orderId}/Chats`
- Slack: `SlackMessageDialog` → `POST /slack/send-message`
- Issues: `RaiseResolveDialog` → Stockship `POST /orders/issueRaise`, `/issueResolved`
- Pending: `GET /dashboard/pending-orders?page=&limit=`
- Pre-Dispatch: `GET /orders/pre-dispatch`
- NDR: `GET /dashboard/ndr-orders?page=&limit=`
- RTO: `GET /orders/rca`
- Failed: `GET /orders/failed`

### Leads
- Lead List: `useLeadsData` → `GET /api/leads-operation/leads-details?start_date=&end_date=&source=&state=&leadProfile=&page=&limit=&contact_status=`
- Analytics: `useLeadsAnalytics` → `GET /api/leads-operation/leads-analytics?start_date=&end_date=`
- Batch KPIs: `useBatchPerformance` → `GET /leads-operation/batch-details`
- Workflows: `useWorkflowsList` → `GET /leads-operation/workflows`
- Workflow Detail: → `GET /leads-operation/workflows/{id}`
- Workflow Dropdowns: → `GET /leads-operation/workflows/data/reference?type=`
- Assignment Logs: → `GET /leads-operation/assignment-logs?start_date=&end_date=&page=&limit=`
- Approved Agents: → `GET /leads-management/approved-agents`

### Agronomy & Training
- Call Audits: `useCallLogs` → `GET /agronomy/call-audits?start_date=&end_date=&page=&limit=`
- Audit Submit: → `POST /agronomy/call-audits/submit`
- Audit Detail: → `GET /agronomy/calls/{call_id}/audit`
- Audit Update: → `PUT /agronomy/call-audits/{audit_id}`
- Audit List: → `GET /agronomy/audit-list?page=&limit=&call_id=&auditor_id=&start_date=&end_date=&status=&auditor=&department=&sla_status=`
- Audit Stats: → `GET /agronomy/call-audits/stats/overview`
- Tests: `useAgronomyTests` → CRM Agronomy API (beta.api.crm.ko-tech.in, auto-login, 13 methods)
- Test Submit: → `POST /agronomy/tests/{test_id}/submit` (auto-evaluate MCQ)
- FCM Notify: → `POST /agronomy/send-test-fcm`
- Question Bank: → `GET /agronomy/question-bank`, `POST /agronomy/question-bank/add`
- Suggestions: `useAgronomySuggestions` → `GET /agronomy/suggestions?page=&limit=`
- Provide Suggestion: → `POST /agronomy/suggestions/provide`
- Principal Cert: → `POST /agronomy/principal-certificate` (form-data + S3 uploads)
- Skill Matrix: `useSkillMatrix` → `GET /agronomy/skill-matrix?email=&department=&page=&limit=`
- Agent Details: `useAgentDetails` → `GET /dashboard/agent-details?agent_email=` (includes call_audits with skill_scores)
- Products: → `GET /agronomy/products`
- SOPs: `useSOPs` → `GET /sop/sops?page=&limit=&search=&tags=&category=`, `POST /sop/sops`, `PUT /sop/sops/{id}`
- New Joining: No backend (UI only, mock data)
- Training Programs: No backend (UI only in SkillMatrix page)

### Reports
- Floor Manager: `useSalesHeadAgents` → `GET /reports/reports&analytics/floor-manager-agents?manager_id=&start_date=&end_date=`
- Manager List: → `GET /reports/reports&analytics/list-managers`
- Sales Head: → `GET /reports/sales-head-agents`

### Admin
- Users: → `GET /auth/users-by-roles`, `POST /auth/assign-role`, `POST /auth/unassign-role`
- Roles: `useModuleAccess` → `GET /auth/module-access`, `POST /auth/update-module-access`

### Pricing — Coupons
- Coupon List: `useCoupons` → `GET /coupons/list?page=&limit=&search=&is_active=`
- Coupon Stats: → `GET /coupons/stats`
- Coupon Detail: → `GET /coupons/{coupon_id}`
- Create Coupon: → `POST /coupons/create`
- Update Coupon: → `PUT /coupons/{coupon_id}`
- Delete Coupon: → `DELETE /coupons/{coupon_id}`
- Usage Logs: → `GET /coupons/usage-logs?page=&limit=&coupon_code=&start_date=&end_date=`
- Rule Metadata: → `GET /coupons/rule-metadata`
- Check Eligibility: → `POST /coupons/check-eligibility`
- Apply Coupon: → `POST /coupons/apply`

### Other
- Profile Tracker: `useProfilingTracker` → Supabase `lead_profiling_trackers`
- Inventory: `useInventory` → `GET /inventory/inventory?page=&limit=&status=`
- Auth: → `POST /auth/login`, `POST /auth/signup`

## External Services
| Service | Pages | Method |
|---------|-------|--------|
| Backend API | ~40 pages | apiClient.ts via Vite proxy |
| Firebase Realtime DB | Dashboard, Reports | REST poll 5s |
| Firebase Firestore | AllOrders (chat) | onSnapshot |
| Firebase Storage | Chat uploads | SDK |
| Stockship | Orders, Inventory | inventoryApi.ts direct |
| Supabase | ProfileTracker | JS client (anon key) |
| CRM Agronomy | Agronomy pages | agronomyApiClient.ts auto-login |
| Slack (via backend) | AllOrders dialog | POST /slack/send-message |

## All localStorage Keys
```
backend_access_token, backend_token_type, backend_user_email,
backend_user_roles, backend_user_department, user_password (base64),
inventory_access_token, inventory_refresh_token,
agronomy_access_token, agronomy_refresh_token, agronomy_token_expiry,
agronomy_agent_id, agronomy_agent_name
```

#active
