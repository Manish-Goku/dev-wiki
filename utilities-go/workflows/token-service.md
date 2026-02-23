# Token Service — Centralized Token Management API

`#active` `#auth-required`

---

## Purpose

Centralized API for backend services to obtain authentication tokens for third-party platforms (Shiprocket, Delhivery, Razorpay). Eliminates token management duplication across services. Two categories: dynamic (refreshable with caching) and static (env-var secrets).

---

## API Reference

### Dynamic Tokens
`GET /api/tokens/dynamic?platform=shiprocket&force_refresh=false`

**Auth:** JWKS JWT middleware
**Query params:**
- `platform` (required) — `shiprocket` or `delhivery-b2b`
- `force_refresh` (optional) — `true` to force refresh (still respects min interval)

### Static Tokens
`GET /api/tokens/static?platform=delhivery-b2c`

**Auth:** JWKS JWT middleware + IP whitelist
**Query params:**
- `platform` (required) — `delhivery-b2c` or `razorpay`

### Response Format

```json
{
  "platform": "shiprocket",
  "auth_type": "bearer",
  "token": "eyJhbGciOiJIUzI1...",
  "expires_at": "2026-02-24T15:00:00Z",
  "cached": true
}
```

| Field | Description |
|-------|-------------|
| `platform` | Platform name |
| `auth_type` | `"bearer"` or `"basic"` — tells client which `Authorization` scheme to use |
| `token` | The token/credential string |
| `expires_at` | ISO 8601 expiry (null for static tokens and Shiprocket) |
| `cached` | true if from cache, false if freshly obtained |

### Error Response

```json
{
  "error": "Invalid platform for this route",
  "valid_platforms": ["shiprocket", "delhivery-b2b"]
}
```

---

## Platform Details

### Shiprocket (Dynamic)

| Aspect | Detail |
|--------|--------|
| Login endpoint | POST `https://apiv2.shiprocket.in/v1/external/auth/login` |
| Credentials | `SHIPROCKET_EMAIL`, `SHIPROCKET_PASSWORD` env vars |
| Response | `{"token": "..."}` |
| Auth type | Bearer |
| Expiry | Not exposed by Shiprocket (validated on use) |
| Cache buffer | 1 minute (no re-refresh within 1 min) |
| Validation | GET `/v1/external/orders` — 401 = invalid |

### Delhivery B2B (Dynamic)

| Aspect | Detail |
|--------|--------|
| Login endpoint | POST `https://ltl-clients-api.delhivery.com/ums/login` |
| Credentials | `DELHIVERY_USERNAME`, `DELHIVERY_PASSWORD` env vars |
| Response | `{"data": {"jwt": "..."}}` |
| Auth type | Bearer (JWT) |
| Expiry | Parsed from JWT `exp` claim |
| Cache buffer | 30 seconds (no re-refresh within 30s) |
| Expiry buffer | 2 hours (refresh if expiring within 2h) |

### Delhivery B2C (Static)

| Aspect | Detail |
|--------|--------|
| Source | `DELHIVERY_AUTH_TOKEN_B2C` env var |
| Auth type | Bearer |
| Expiry | None (static) |
| IP restricted | Yes (via `ALLOWED_IPS`) |

### Razorpay (Static)

| Aspect | Detail |
|--------|--------|
| Source | `RAZORPAY_KEY_ID` + `RAZORPAY_KEY_SECRET` env vars |
| Encoding | Base64(`key_id:key_secret`) |
| Auth type | Basic |
| Expiry | None (static) |
| IP restricted | Yes (via `ALLOWED_IPS`) |
| Client usage | `Authorization: Basic {token}` |

---

## Thread-Safe Token Managers

### RefreshableTokenManager (Shiprocket)

**File:** `utils/token_manager.go`

Generic token manager with pluggable refresh function:

```
RefreshToken(client, minRefreshInterval, forceRefresh)
    │
    ├── Refreshed within minRefreshInterval? → return cached
    │
    ├── Another goroutine refreshing? → wait (sync.Cond)
    │
    ├── After wait, re-check interval → return cached if refreshed
    │
    └── Call refreshFunc(client) outside lock → cache result → broadcast
```

### DelhiveryB2BTokenManager (Delhivery B2B)

**File:** `utils/token_manager.go`

Specialized for JWT tokens with expiry checking:

```
RefreshTokenIfNeeded(client, minRefreshInterval, expiryBuffer)
    │
    ├── Check 1: refreshed within minRefreshInterval? → cached
    │
    ├── Check 2: JWT exp > now + expiryBuffer? → cached (still valid)
    │
    ├── Check 3: another goroutine refreshing? → wait
    │
    └── Refresh via DelhiveryNewTokenWithHeaders() → cache → broadcast
```

Both managers use `sync.Cond` to prevent thundering herd — when multiple goroutines need a token simultaneously, only one performs the actual refresh.

---

## IP Whitelist Middleware

Applied to `/api/tokens/static` only.

**File:** `handlers/token_handler.go`

- Reads `ALLOWED_IPS` env var (comma-separated)
- `0.0.0.0` → allow all IPs
- Empty → allow all (with log warning)
- Non-matching IP → 403 Forbidden

---

## Token Usage by Other Modules

| Token | Used By | Via |
|-------|---------|----|
| Shiprocket | Order tracking (Shiprocket tracker) | `GetShiprocketToken()` (validate-then-refresh) |
| Delhivery B2B | Order tracking (B2B tracker) | `B2BTokenManager.RefreshToken()` |
| Delhivery B2C | Order tracking (B2C + fallback trackers) | `DELHIVERY_AUTH_TOKEN_B2C` env var directly |
| SSO | Cron jobs HTTP execution | `SSOTokenManager.GetToken()` (separate manager, not in token API) |

Note: The token API and internal tracking use **different code paths** to the same token managers. The API provides external access; tracking uses the managers directly.

---

## Environment Variables

| Var | Platform | Purpose |
|-----|----------|---------|
| `SHIPROCKET_EMAIL` | Shiprocket | Login email |
| `SHIPROCKET_PASSWORD` | Shiprocket | Login password |
| `DELHIVERY_USERNAME` | Delhivery B2B | Login username |
| `DELHIVERY_PASSWORD` | Delhivery B2B | Login password |
| `DELHIVERY_AUTH_TOKEN_B2C` | Delhivery B2C | Static Bearer token |
| `RAZORPAY_KEY_ID` | Razorpay | API key ID |
| `RAZORPAY_KEY_SECRET` | Razorpay | API key secret |
| `ALLOWED_IPS` | Static endpoint | Comma-separated IP whitelist (0.0.0.0 = all) |

---

## Files

| File | Purpose | Lines |
|------|---------|-------|
| handlers/token_handler.go | Dynamic/static handlers, IP whitelist, platform routing | 307 |
| utils/token_manager.go | RefreshableTokenManager, DelhiveryB2BTokenManager, TokenCache | 320 |
| utils/token_helper.go | Shiprocket login, Delhivery B2B login, token validation | 201 |
| routes/tokenRoutes.go | Route registration (2 endpoints) | 28 |
