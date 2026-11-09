# P: Personalia integration endpoints (built in `kpi-personalia-api`)

**Purpose:** The Personalia half of the SiAPP seam: a small, server-to-server API that SiAPP calls with a shared key. SiAPP calls Personalia; Personalia never calls SiAPP through this API (webhooks are a separate seam, S6).

**Repository:** `kpi-personalia-api` (Go, `/api/v3`), the rewrite of the Laravel API in `PJ48-KPI-Personalia-AdminPanel`. Both read and write **the same MySQL database** (`m_users`, `m_employee_work`, `t_employee_device`, …; Laravel owns the migrations). Building here instead of in Laravel means nothing has to be ported again when the Laravel API is retired. Follow that repo's conventions (`internal/modules/<name>/{handler,service,dto,errors}.go`, sqlc queries in `db/query/`, config in `configs/config.example.yaml`), not SiAPP's.

**Needed by:** SiAPP 01.1 (P.1), 01.2 and 01.3 (P.2). **P.1 and P.2 are built in SiAPP Phase 00.5 (Part A)**, together with the `m_users.domain` column and this repo's new migration system. Where this file and 00.5 differ, 00.5 wins: in particular **the integration login does no device binding** (SiAPP binds phones itself) and `Identity` carries `domain` instead of `device_token_sha256`.

**Why not Personalia's normal login** (`POST /api/v3/auth/login`):
- It rate-limits per IP. Every SiAPP login arrives from SiAPP's server IP, so a few logins would lock everyone out.
- It issues Personalia tokens and revokes the user's other Personalia sessions. SiAPP needs neither.
- Its protected routes require a device-signed request (`VerifyDevice`), which a server cannot produce.
- Verifying Personalia's own JWTs inside SiAPP would mean sharing `jwt.secret` (HS256), and anyone holding it can mint a token for any user.

**Organization mapping (D11):** SiAPP's organization is the employing company, `m_employee_work.company_id` of the person's latest work row (a row of `m_companies`). The placement client (`client_id`) is returned alongside for display and for project mapping, but is not the tenant.

---

## Phase P.1: Integration key and login

**Goal:** SiAPP can check an email and password and get the person's identity, without a Personalia token being issued and without touching the person's Personalia phone binding.

**Build**
- **Config** (`configs/config.example.yaml`, new section):
  ```yaml
  integrations:
    siapp:
      # SHA-256 hex of each accepted key. Two entries allow rotation without downtime.
      # Empty list → the /integrations/siapp routes are not mounted (404).
      key_hashes: []
      # Failed logins allowed per email per 15 minutes before 429.
      login_failures_per_email: 10
  ```
- **`middleware.RequireIntegrationKey(keyHashes)`**: reads `X-Integration-Key`, hashes it with SHA-256, compares against each configured hash with `subtle.ConstantTimeCompare`. Missing or wrong → 401. Never logs the header. Comment why only hashes are in config: a leaked config file does not leak a usable key.
- **Router:** a new group `/api/v3/integrations/siapp` mounted **outside** the `Authenticate` / `VerifyDevice` groups, with only `RequireIntegrationKey`. Comment why: the caller is a server, not a user's phone, so user tokens and device signatures do not apply; the key is the authentication.
- **Module `internal/modules/integration`**
  - ⚠ Extract the credential check from `auth.Service.Login` into one function both use (`auth.Service.VerifyCredentials(ctx, email, password) (domain.User, domain.EmployeeWork, error)`): email lookup, the dummy-hash compare for unknown emails (constant time), bcrypt compare, latest employee work, `IsEmployeeInactive`. Comment why: the integration login must never drift from the app login's rules (for example a resigned employee getting into SiAPP).
  - **`POST /api/v3/integrations/siapp/login`** `{ email, password, user_agent?, client_ip? }`
    1. `VerifyCredentials`. Wrong email or password → 401 `invalid_credentials` (one response for both).
    2. No employee work row → 403 `employee_work_missing`; inactive → 403 `employee_inactive`.
    3. Return `Identity` (below). Accounts of both domains (`personalia`, `siapp`) are accepted. **Do not** issue tokens, **do not** call `RevokeAllRefreshTokensForUser`, **do not** touch `t_employee_device` or FCM tokens: phone binding for SiAPP lives in SiAPP, so the Personalia app and SiAPP never block each other.
    - ⚠ Per-email failure limit (`login_failures_per_email` per 15 min) → 429. Comment why: requests all come from SiAPP's IP, so the per-IP limiter cannot see a guesser spread across many IPs; SiAPP limits per IP on its side, and this limit protects the account itself.
    - Log `client_ip` and `user_agent` (sent by SiAPP) in the auth-failure log so HR sees the real phone, not SiAPP's server. Never log the password.
- **`Identity`** (JSON, also used by P.2):
  ```json
  {
    "user_id": 123,
    "email": "budi@contoh.com",
    "nik": "0012345678",
    "full_name": "Budi Santoso",
    "phone": "0812…",          // null when unknown
    "photo_url": "https://…",  // null when unknown
    "company": { "id": 7, "name": "PT …" },
    "client":  { "id": 41, "name": "PT …" },
    "position": { "id": 3, "name": "Satpam" },
    "active": true,
    "domain": "siapp"  // "personalia" (Personalia + SiAPP) | "siapp" (SiAPP only)
  }
  ```
- **Admin panel:** HR sets `domain` in the user form (SiAPP 00.5 Part B). HR's device reset stays Personalia-only.

**Tests**
- No key, wrong key → 401; a rotated second key works; empty `key_hashes` → 404.
- Correct password → identity; wrong password and unknown email → the same 401; resigned employee → 403 `employee_inactive`; no work row → 403.
- `t_employee_device` is unchanged by any integration login; both domains log in.
- The login does not create a `t_refresh_tokens` row and does not revoke existing ones.
- The 11th failure for one email within 15 minutes → 429, whatever the `client_ip`.
- `auth.Service.Login` tests still pass after the extraction.

---

## Phase P.2: Employee lookup

**Goal:** SiAPP can fetch people by NIK for team management and for its re-validation and daily resync.

**Build**
- **`POST /api/v3/integrations/siapp/employees`** `{ niks: ["…"] }` (1–200 NIKs) → `{ data: [Identity…] }`.
  - Same `Identity` shape as P.1, built from the latest `m_employee_work` row per NIK.
  - NIKs that do not exist are **omitted**, not errors. Inactive employees are returned with `active: false`, so SiAPP can tell "deactivated" from "typo".
  - One query per batch (`WHERE employee_nik IN (…)` plus a latest-work join), not one per NIK. Comment the N+1 risk.
- It is a `POST` only so 200 NIKs fit in a body; it changes nothing.

**Tests**
- Known, unknown and inactive NIKs in one batch → two items, one with `active: false`.
- 201 NIKs → 422.
- A person moved to another company returns the new `company.id`.
- `domain` is returned for every person.

---

## Later seams

The other seams in `docs/plan/README.md` (S2 org read, S3 attendance push; S5 leave is retired, SiAPP owns leave in 07.5) go in the same `/integrations/siapp` group with the same key, one phase each, when the SiAPP module that needs them is next. S6 (webhooks) is the one call that goes from Personalia to SiAPP and uses a signature instead (SiAPP 05.3).
