# Order Pipeline & Dashboard

## Order Status Flow
```
 1: ready_to_confirm → 2: ready_to_invoice → 3: ready_to_book → 4: ready_to_pick
 → 5: ready_to_pack → 6: ready_to_check → 7: ready_to_racked → 8: ready_to_manifest
 → 9: dispatched → 14: in transit → 25: out for delivery → 15: delivered

Branches:
 → 52: ndr (non-delivery) → retry or → 11: rto
 → 16: rto in transit → 17: rto received
 → 18: lost → 19: destroy
 → 10/-1/-5: cancelled
 → 0: failed
 → 55: picked
 → 20-23: not answered (1-4 attempts)
 → 26: pending
 → 30: ambiguous
```

## Dashboard Endpoints
| Endpoint | Filter | Purpose |
|----------|--------|---------|
| `GET /dashboard/orders-summary` | date range + status + payment | KPIs + paginated orders |
| `GET /dashboard/pending-orders` | page, limit | Support pending (is_mistake.status=true) |
| `GET /dashboard/order-details/{id}` | — | Full order detail |
| `GET /dashboard/others-retailers-orders` | date, status, payment, category, search | All orders list |
| `GET /dashboard/pre-dispatch-confirmation` | page, limit | Pre-dispatch (status=4) |
| `GET /dashboard/ndr-orders` | page, limit | NDR (status=52) |
| `GET /dashboard/escalated-orders` | page, limit | Escalated orders |
| `GET /dashboard/failed-orders` | page, limit | Failed orders |
| `POST /dashboard/mark-order-resolved` | order_id, category, message | Clear is_mistake.status |
| `GET /dashboard/invoice-url/{order_id}` | — | Get Zoho PDF invoice URL |

## SLA Calculation
- **Duration**: 4 working hours from creation
- **Working hours**: 10 AM - 7 PM IST, Mon-Sat
- **Frontend**: `lib/slaCalculator.ts` — `isSLABreached()`, `formatRemainingTime()`
- **Backend**: `calculate_sla_time()` in `dashboard/route.py`

## Support Pending Detection
Order is "support pending" when:
1. `is_mistake.status = true`
2. `is_mistake.chats` references chat doc with `department: "dispatch"` + `raised_by` non-empty
3. Excludes chats with `department: "support"` + `raised_by` non-empty

## Frontend Pages
| Page | Route | Hook | Data Source |
|------|-------|------|------------|
| Dashboard | `/` | useDashboardSummary | Backend |
| All Orders | `/orders/all` | useRetailerOrders | Backend |
| Support Pending | `/orders/pending` | — | Backend |
| Pre-Dispatch | `/orders/pre-dispatch` | usePreDispatchOrders | Backend |
| NDR | `/orders/ndr` | useNDROrders | Backend |
| RTO | `/orders/rto` | useRCAOrders | Backend |
| Failed | `/orders/failed` | useFailedOrders | Backend |

## Invoice Download
- **Backend**: `GET /dashboard/invoice-url/{order_id}` queries `invoices` collection for matching `order_id` with `invoice_url` present
- **URL Transform**: `_generate_zoho_pdf_url()` converts Zoho secure URL to direct PDF URL by extracting `CInvoiceID` param and building `/api/v3/clientinvoices/secure?CInvoiceID=...&accept=pdf`
- **Frontend**: "Download Invoice" option in AllOrders 3-dot dropdown menu, opens PDF in new tab
- **Collection**: `invoices` (shared with Inventory-Management-Backend), fields: `order_id`, `invoice_url`, `invoice_number`

## Order Detail Dialog
Fetches: `useOrderDetails(orderId)` + `useStockshipOrderTimeline(orderId)`
Shows: customer info, items, timeline events, chat history, tracking URL

## Tracking URLs
```
Delhivery:  https://www.delhivery.com/track-v2/package/{awb}
Shiprocket: https://shiprocket.co/tracking/{awb}
Velocity:   https://velexp.com/track/{awb}
```

#active
