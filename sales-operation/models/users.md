# users

**Collection**: `users` (in CRM-Database)
**Used by**: `auth/` module

## Schema
```javascript
{
  _id: ObjectId,
  email: String (unique),
  password: String,                    // bcrypt hash
  name: String,
  dpt_name: String,                    // department name
  userRoles: [
    {
      roleName: String,                // "sales_admin", "floor_manager", etc.
      isAssigned: Boolean              // false = revoked (not deleted)
    }
  ],
  otp: null,
  otpExpirationTime: null,
  otpVerified: Boolean,
  isfranchise: Boolean,
  refreshToken: String | null,
  isActive: Boolean
}
```

## Allowed Roles (for login)
`sales_admin, sales_head, floor_manager, support_manager, junior_support, senior_agronomy, junior_agronomy, ba_team`

Default signup role: `retention` — NOT in allowed list, cannot login.

## Auth Flow
1. `POST /auth/login` → bcrypt verify → extract `userRoles[].roleName` where `isAssigned=true`
2. Check at least one role in ALLOWED_ROLES
3. Generate JWT: `{ sub: email, roles: [...] }`, HS256, 480min expiry
4. Return: `{ access_token, token_type: "bearer", email, roles, dpt_name }`

## Role Assignment
- `POST /auth/assign-role` (sales_admin only) — adds role with `isAssigned: true`
- `POST /auth/unassign-role` — sets `isAssigned: false` (keeps entry)
- Returns: `"newly_assigned" | "already_assigned" | "reactivated"`

#active #auth-required
