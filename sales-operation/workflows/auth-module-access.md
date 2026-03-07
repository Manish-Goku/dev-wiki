# Auth & Module Access Flow

## Login Flow
```
1. Frontend: POST /auth/login { email, password }
2. Backend: bcrypt verify → extract assigned roles → check ALLOWED_ROLES
3. Backend: generate JWT (HS256, 480min) → { access_token, token_type, email, roles, dpt_name }
4. Frontend stores in localStorage:
   - backend_access_token, backend_token_type ("bearer")
   - backend_user_email, backend_user_roles (JSON array)
   - backend_user_department, user_password (base64)
5. Frontend pre-fetches Stockship token (inventoryLogin)
6. Frontend redirects based on role:
   floor_manager → /reports/floor-manager
   support_manager, junior_support → /orders/all
   others → /home-blank or /
```

## Module Access Flow
```
1. AuthContext mounts → checks localStorage for backend_access_token
2. Creates synthetic User { id: email }
3. GET /auth/module-access → all role→module mappings
4. Matches first role from localStorage against API response
5. Stores userModuleAccess in context
6. AppSidebar filters nav items by userModuleAccess.modules
7. If no module access data → show all (fallback)
```

## JWT Structure
```json
Header:  { "alg": "HS256", "typ": "JWT" }
Payload: { "sub": "user@ko.com", "roles": ["floor_manager", "sales_admin"], "exp": 1709136000 }
Secret:  SECRET_KEY env var (default: "your-secret-key-change-this-in-production")
```

## Backend Auth Dependencies
```python
get_current_user(token)              # JWT decode → user dict
require_sales_admin_role(user)       # "sales_admin" only
require_retention_role(user)         # "retention" only
require_pbi_or_retention(user)       # any ALLOWED_ROLE — used by most routes
```

## Secondary Auth Systems
| System | Purpose | Auth Method |
|--------|---------|-------------|
| CRM Agronomy API | Tests, question bank | Auto-login: sahilsoni@katyayaniorganics.com / Sahil@123 |
| Stockship API | Inventory, orders | User's own email/password (from localStorage) |
| Supabase | Profiling tracker | Anon key (no user auth) |

## Token Expiry
- Backend JWT: 480 min (8 hours)
- No refresh token flow — user re-logins
- Frontend: on 401 → clears localStorage → redirect /auth

#active #auth-required
