# 01: Identity and login

**Purpose:** Sign people in **with their Persist Personalia account** (email + password, checked by Personalia), issue SiAPP's own short-lived tokens, manage sessions and devices, and keep a credential-free cache of the people SiAPP works with.

**LOCK:** F2 (SSO / Auth): "Tak ada user store sendiri." NIK is the cross-product key (LOCK/01 §3). ⚠ The LOCK's "OAuth2 client ke Laravel Passport" does not match the real Personalia: Passport is installed but switched off, and its API uses `tymon/jwt-auth`. The seam is therefore a **server-to-server credential check** with a shared integration key, not OAuth (D8 in `docs/decisions.md`).

**Manual refs:** §2.1 Login, §3.2.1, §3.2.1.3 Ingat saya.

**Needs:** 00. Personalia side: [`personalia-integration.md`](personalia-integration.md) P.1 (login) and P.2 (employee lookup).

**Contract operations:** `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me`, `POST|DELETE /notifications/token`.

**Error reasons:**
- `AUTH_INVALID_CREDENTIALS` (code `UNAUTHENTICATED`)
- `AUTH_ACCOUNT_DISABLED` (code `FORBIDDEN`)
- `AUTH_DEVICE_BOUND` (code `FORBIDDEN`)
- `AUTH_DEVICE_CHANGED` (code `UNAUTHENTICATED`)
- `AUTH_ORGANIZATION_SUSPENDED` (code `FORBIDDEN`)
- `AUTH_TOKEN_INVALID`, `AUTH_TOKEN_EXPIRED` (codes `UNAUTHENTICATED`, `TOKEN_EXPIRED`)
- `AUTH_REFRESH_REUSED` (code `UNAUTHENTICATED`)
- Admin (`pkg/access`, Phase 01.3): `NOT_ADMIN` (code `FORBIDDEN`; also the web login gate and a WEB refresh without a web role), `ACCESS_ORG_ADMIN_ONLY` (code `ROLE_DENIED`), `ACCESS_OUTSIDE_ADMIN_SCOPE` (code `SCOPE_DENIED`), `ACCESS_OWN_ROLE_PROTECTED` (code `CONFLICT`)

**Lock order (every phase of this module):** a transaction that changes a person's sessions, phone binding or devices locks the person's `personnels` row **first** (`LockPersonnel`; login takes it through its upsert), then sessions, then the binding. One order means no deadlocks, and a logout or reset that waits for an in-flight refresh sees the session that refresh created.

**What stays in Personalia** (do not build): passwords and their policy, the default birth-date password (`DDMMYYYY`), forgot-password and reset links, change-password, creating and disabling accounts, the account's `domain` (HR sets it in the Personalia admin panel), login help FAQ, and all email. **The one-phone binding is not Personalia's:** SiAPP keeps its own (`device_bindings`), because the integration login never touches Personalia's device table (Phase 00.5). HR's "reset device" in the Personalia admin panel frees the Personalia app only; SiAPP has its own reset (Phase 01.3). The mobile and web clients link to Personalia for these.

**How login works**
```
SiAPP app ──(email, password, device_token)──▶ SiAPP POST /auth/login
                                                    │  X-Integration-Key
                                                    ▼
                         Personalia POST /api/v3/integrations/siapp/login
                           checks password + active employee,
                           issues NO token, binds NO device
                                                    │
              { user_id, nik, name, company, client, active, domain, ... }
                                                    ▼
                         SiAPP upserts the organization and the person,
                         binds the phone (device_bindings, SiAPP's own),
                         then issues its own access + refresh tokens
```
SiAPP never receives or verifies a Personalia token. The only secret shared between the two systems is the integration key (`PERSONEL_INTEGRATION_KEY`, Phase 00.5).

**What changed from the first plan:**
- No `users` table. People are rows in `personnels`, a cache of Personalia data with no credentials.
- The token's `sub` is the local `personnels.id`; it also carries `organization_id` and the session id.
- The organization is the person's **employing company** in Personalia (`m_employee_work.company_id`, D11), never a value from the request.
- One bound device per person, enforced by SiAPP's own `device_bindings` table. Personalia's device table is not involved: the Personalia app and SiAPP never block each other on one phone.
- Sessions, refresh rotation, reuse detection, the revocation list and the default-deny middleware are kept from the first build.

---

## Phase 01.1: Login, people cache and SiAPP tokens 🔌

**Goal:** A person with a Personalia account gets a SiAPP token pair. **Every route requires a valid SiAPP access token except the public allowlist.**

**Build**
- **Contract changes (PR to `persist-ecosystem`):** `LoginRequest` gains `device_token` (string, required when `client = MOBILE`) and `client` (`MOBILE` | `WEB`). `identifier` must be an email: describe it as "email akun Personalia". `organization_code` is ignored (the organization comes from Personalia); mark it deprecated.
- **Table `personnels`** (tenant table, RLS)
  - `id`, `organization_id`
  - `personel_user_id` (BIGINT, Personalia's `m_users.id`), UNIQUE (`organization_id`, `personel_user_id`)
  - `nik` (VARCHAR(100), Personalia's own width; nullable, D4), partial UNIQUE (`organization_id`, `nik`) `WHERE nik IS NOT NULL`. ⚠ Comment that NIK is stored as a string (leading zeros, DL-18) and is the key Personalia recognises.
  - `client_id` (BIGINT, nullable), `client_name` (nullable): the Personalia client the person is placed at, for display and for module 02's project mapping.
  - `full_name`, `email` (nullable), `phone` (nullable), `photo_url` (nullable)
  - `status` ENUM(`ACTIVE`,`INACTIVE`)
  - `synced_at`, `last_seen_at` (nullable), timestamps
  - **No password, no birth date.** Comment why: LOCK F2.
- **Table `sessions`** (tenant table, RLS)
  - `id`, `organization_id`, `personnel_id` (FK), `family_id`
  - `refresh_token_hash` (CHAR(64), UNIQUE)
  - `device_token_hash` (CHAR(64), nullable; NULL for `WEB`): SHA-256 hex of the device token this session logged in with. 01.2 compares it with the person's current binding to notice that the binding was reset or moved.
  - `remember_me`, `client` ENUM(`MOBILE`,`WEB`)
  - `expires_at`, `revoked_at`, `replaced_by_id`, `user_agent`, `ip`, `last_used_at`, timestamps
- **Table `device_bindings`** (tenant table, RLS): the one phone a person may use SiAPP on.
  - `id`, `organization_id`, `personnel_id` (FK), UNIQUE (`organization_id`, `personnel_id`): at most one binding per person
  - `device_token_hash` (CHAR(64)): SHA-256 hex of the device token. Plain tokens are never stored.
  - `bound_at`, `last_seen_at`, `reset_at` / `reset_by_personnel_id` (nullable: set on the audit trail of a reset, the row itself is deleted)
  - ⚠ Comment why this is SiAPP's and not Personalia's: the integration login never reads or writes `t_employee_device`, so the Personalia app and SiAPP bind independently.
- **`auth.Tokens`** (keep from the first build)
  - Access JWT (HS256, 15 min) with claims `sub` (personnel id), `organization_id` (the contract's claim name), `sid`, `iat`, `exp`. Secret from `JWT_SECRET`.
  - Refresh token: 32 random bytes, base64url, prefixed with the organization id: `"{orgID}.{random}"`. Only the SHA-256 hex of the whole string is stored.
  - ⚠ Comment why the organization id is in the refresh token: refresh runs before any tenant is known, and the prefix lets the lookup run inside `WithTenant` for that organization instead of scanning across tenants. The prefix is not trusted on its own: the hash must match a row inside that tenant.
- **`identity.Login(ctx, LoginInput{Email, Password, DeviceToken, Client, RememberMe, IP, UserAgent})`** ⚠ the single path that starts a session:
  1. `personel.Auth.Login` (Phase 00.5) → `Identity`. It sends no device token: Personalia does not know about devices. Map its answers:
     - `ErrInvalidCredentials` → `AUTH_INVALID_CREDENTIALS`;
     - `ErrAccountDisabled` (inactive, or no employee work row) → `AUTH_ACCOUNT_DISABLED`;
     - `ErrRateLimited` → `RATE_LIMITED` (429);
     - `ErrUnavailable` → `PERSONEL_UNAVAILABLE` (503, `apperr.ErrPersonelUnavailable`).
     - Both `Domain` values (`personalia`, `siapp`) may sign in to SiAPP.
  2. Upsert `organizations` (`WithoutTenant`) with the company id and name: first sight inserts it as `ACTIVE`. A `SUSPENDED` organization → `AUTH_ORGANIZATION_SUSPENDED`.
  3. In `WithTenant(identity.OrganizationID)`: upsert `personnels` by `personel_user_id` (`ON CONFLICT … DO UPDATE` with the fresh name, NIK, client, phone, photo, status and `synced_at`).
  4. For `MOBILE`, in the same transaction, lock the person's row and apply the binding rule:
     - no binding → create it from the device token;
     - same hash → update `last_seen_at`;
     - another hash → `AUTH_DEVICE_BOUND`, message "Akun Anda terhubung dengan perangkat lain. Hubungi admin untuk ganti perangkat.", and no session is created.
     The binding is created only after Personalia has confirmed the password and an active employee, so a guesser cannot claim a phone slot.
  5. Create the session (with `device_token_hash` for `MOBILE`) and return the pair.
  - Comment the order: the organization row must exist before any tenant insert, because of the FK.
  - ⚠ Comment the device rule: the person may have only one bound phone, and `device_bindings` is the single record of it. A lost or replaced phone is freed by an admin through SiAPP's own reset (Phase 01.3). `WEB` logins bind nothing (a browser has no stable device id); Phase 01.3 restricts web login to people with a web role, so field staff cannot use the web to get around the one-phone rule.
- **`POST /auth/login`** `{ identifier, password, device_token?, client, remember_me? }` → `LoginResult`. **Public.** (D3)
  - ⚠ SiAPP never stores, logs or hashes the password; it only passes it to Personalia over TLS. Comment this and the reason (people sign in to SiAPP's own login screen with their Personalia account, manual §2.1 and D3).
  - Every credential failure returns the same `AUTH_INVALID_CREDENTIALS` response, so accounts cannot be enumerated.
  - Keep the failure rate limits from the first build: 10 failed logins per minute per IP and identifier, 100 per minute per IP. Personalia's per-email failure limit (P.1) is the defence against guesses spread over many IPs.
- **`auth.Require` middleware (default-deny)** — keep from the first build, with these changes:
  - Puts `auth.Principal{PersonnelID, OrganizationID, SessionID}` into the context.
  - ⚠ Every later `WithTenant` call takes the organization from `auth.From(ctx)`, never from a path, query or body. Comment this as the tenancy rule (LOCK principle 2).
  - `auth.PublicRoutes`: `GET /api/v1/health`, `POST /api/v1/auth/login`. Later phases add their entries.
  - Unknown paths return 401 without a token (not 404).
- **`cmd/seed`**: seeds nothing for identity (people come from Personalia). In dev it prints the fixture emails and password of the real `kpi-personalia-api` seed (`db/seed/40_siapp.sql`, the same ones `personeltest` mirrors).

**Tests**
- Unit: token round-trip; refresh-token prefix parsing rejects a malformed value.
- HTTP (against the `personeltest` fake)
  - Login works for a fixture person, of either domain; the person row is created; a second login updates the name.
  - A wrong password returns the generic error; the 11th failure for one identifier from one IP returns 429.
  - `MOBILE` without `device_token` → `VALIDATION_ERROR`. The first phone binds. A second phone for the same person → `AUTH_DEVICE_BOUND`, no session is created, and the first phone still logs in. A wrong password never binds a phone. `WEB` login binds nothing.
  - The fake's `Calls()` shows exactly one login request per attempt, and the request body carries no device token.
  - An inactive person → `AUTH_ACCOUNT_DISABLED`; a suspended organization → `AUTH_ORGANIZATION_SUSPENDED`; the fake `Down()` → 503 `PERSONEL_UNAVAILABLE`.
  - People with the same NIK in two organizations get two separate rows, and each token only sees its own.
  - A protected route returns 401 without a token, and 401 for malformed, expired and wrongly signed tokens.
  - **Guard test** (keep): walk every route with `chi.Walk` and require 401 without a token unless it is in `auth.PublicRoutes`; and require that the contract's `security: []` operations equal `auth.PublicRoutes`.
  - The response and logs never contain the password, the device token or the integration key.

---

## Phase 01.2: Refresh, logout and sessions

**Goal:** Refresh tokens rotate securely, reuse is detected, and ending a session cuts off its access tokens at once. (Adapted from the first build.)

**Build**
- **`POST /auth/refresh`** `{ refresh_token }` → `LoginResult`. **Public.**
  - Parse the organization prefix, then run everything in `WithTenant(org)` in one transaction.
  - Select the session `FOR UPDATE` by hash. ⚠ Comment why: two concurrent refreshes with the same token must not both succeed.
  - Revoke the old row (`revoked_at`, `replaced_by_id`) and insert the new one with the same `family_id`, `remember_me`, `client` and `device_token_hash`.
  - The person's current status is re-read: `INACTIVE` → `AUTH_ACCOUNT_DISABLED`.
  - ⚠ **Device check, on every refresh of a `MOBILE` session** (a local read, no Personalia call): the person's `device_bindings` row must exist and its hash must equal the session's `device_token_hash`. Otherwise (reset, or another phone bound since) revoke the whole family and return `AUTH_DEVICE_CHANGED`. The person signs in again, which binds the phone they now use.
  - ⚠ **Web role check, on every refresh of a `WEB` session** (added in 01.3): a person whose web role was revoked has the whole family revoked and gets `NOT_ADMIN`, so a browser session cannot outlive the web login gate.
  - ⚠ **Re-validation with Personalia:** if `personnels.synced_at` is older than `personelResyncAfter = 12 * time.Hour`, refresh the person through `personel.Directory.EmployeesByNIKs` before issuing tokens:
    - inactive, gone, or now employed by another company → `AUTH_ACCOUNT_DISABLED`.
    - If Personalia is unavailable, allow the refresh and keep the cached status.
    - Comment the trade-off: a person deactivated in Personalia keeps access for at most 12 h plus one access-token lifetime, unless the Personalia webhook (05.3) arrives sooner. A device reset needs no such delay: it is SiAPP's own data and takes effect at the next refresh (and at once, because the reset also revokes the person's `MOBILE` sessions, 01.3).
- **Revoke reason** (added after PR #2 review): every revoke records why in `sessions.revoke_reason` (`ROTATED`, `LOGOUT`, `REUSE_DETECTED`, `DEVICE_RESET`, `DEVICE_CHANGED`, `ACCOUNT_DISABLED`, `WEB_ROLE_REVOKED`; a CHECK makes it mandatory). A refresh with a revoked token answers by that reason: only `ROTATED` is a replay; a device reset gives `AUTH_DEVICE_CHANGED`, a deactivation `AUTH_ACCOUNT_DISABLED`, a logout `AUTH_TOKEN_INVALID`.
- **Reuse detection** (⚠, keep): a token that is already revoked revokes **every** session in its family and returns `AUTH_REFRESH_REUSED`. The family revoke must commit before the error is returned.
- **`POST /auth/logout`** `{ firebase_token? }`: revokes the current session's chain and deletes its device (01.3). It does **not** unbind the phone: the binding is the admin's record and outlives a logout.
- **Revocation list** (keep): ending a session writes its id to Redis `revoked-session:<id>` for `accessTokenTTL + 1 min`; `auth.Require` checks it. Fails open when Redis is down (comment kept).
- **Sessions for web.** Web clients go through the Next.js BFF, which stores the pair in an httpOnly cookie (DL-52). Nothing changes in the API; comment on `client = WEB` that the BFF, not the browser, holds the refresh token.
- **`cmd/worker`**: asynq server and scheduler. Periodic task `auth:cleanup-sessions` (daily, 20:00 UTC), run **per organization** in `WithTenant`: delete sessions revoked or expired more than 30 days ago, in batches of 1000.

**Tests**
- After a rotation, the old token is rejected; reusing it kills the whole family.
- Two parallel refreshes with the same token: exactly one succeeds (and removing `FOR UPDATE` makes the test fail).
- A refresh token with another organization's prefix finds nothing (`AUTH_TOKEN_INVALID`) and touches no rows.
- A stale person is re-synced on refresh; the fake `Deactivate`s them → refused.
- The binding is reset and another phone logs in → the first phone's next refresh returns `AUTH_DEVICE_CHANGED` and its family is revoked. The same happens when the binding row is simply gone.
- The device check does not call Personalia: with the fake `Down()`, a refresh from the bound phone still succeeds.
- After logout, the old access token gets 401 everywhere. Redis down: valid tokens still pass (fail open).

---

## Phase 01.3: Profile, devices, web roles and people sync

**Goal:** `GET /auth/me` for menus and profile, push-token registration, the admin gate for web users, the admin's device reset, and keeping the people cache fresh.

**Build**
- **Table `web_role_assignments`** (tenant table, RLS)
  - `id`, `organization_id`, `personnel_id` (UNIQUE within the organization)
  - `role` ENUM(`ORG_ADMIN`,`PROJECT_ADMIN`)
  - `project_ids` BIGINT[] (required for `PROJECT_ADMIN`, empty for `ORG_ADMIN`; CHECK constraint)
  - `granted_by_personnel_id` (nullable), timestamps
  - ⚠ Comment why this lives in SiAPP even though identity is Personalia's: authorization inside the Security add-on ("RBAC scope-project", LOCK/03 §1) is the add-on's own data. Personalia says *who* someone is; SiAPP says what they may do here.
- **`access.RequireAdmin`** middleware (the package `access` starts here; Phase 03 adds field permissions): loads the caller's assignment, rejects with `FORBIDDEN` (reason `NOT_ADMIN`) when there is none, and stores `access.AdminScope{ All bool, ProjectIDs []int64 }` in the context. Admin usecases filter by it. Mounted once on the `/api/v1/admin` group.
- ⚠ **Web login gate:** `identity.Login` with `client = WEB` now refuses a person without a web role assignment (`FORBIDDEN`, reason `NOT_ADMIN`), before any session is created. Comment why: web logins bind no device (01.1), so without this gate field staff could sign in on the web and get around the one-phone rule.
- **Device reset (admin):** `DELETE /admin/personnel/{personnel_id}/device-binding` (a contract change, since HR's reset lives in Personalia and does not reach SiAPP). `ORG_ADMIN`, or a `PROJECT_ADMIN` for a person placed at one of their projects.
  - In one transaction: delete the person's `device_bindings` row and revoke their `MOBILE` sessions (and write their ids to the revocation list), so the old phone is out at once.
  - Audit `device_binding.reset` with the admin as actor. A person with no binding is a no-op answered 200 with `"data": null` (the `docs/api-response.md` rule for deletes and actions).
  - Comment that this is separate from the Personalia admin panel's reset on purpose.
- **Admin endpoints** (contract change: these replace `/admin/web-users`, which is Personalia-owned under D5):
  - `GET /admin/web-roles`, `PUT /admin/web-roles/{personnel_id}` `{ role, project_ids }`, `DELETE /admin/web-roles/{personnel_id}`. `ORG_ADMIN` only.
  - An admin cannot remove or narrow their own assignment (`CONFLICT`), so the last admin cannot lock everyone out.
- **Bootstrap:** `cmd/seed grant-admin --org <id> --nik <nik>` creates the first `ORG_ADMIN` of an organization (runs as the owner role inside `WithTenant`; fetches the person through `identity.People.Ensure` if they have never signed in). Comment that this is the only way to create an admin without an existing one.
- **Audit:** `web_role.grant`, `web_role.revoke`.
- **`GET /auth/me`** → `MeResult`: the person (NIK masked), the organization, and — once Phase 02.2 exists — the person's active team memberships with their role. Before 02.2 the memberships list is empty. It also returns `web_role` (this phase) and, once Phase 03.1 exists, `permissions` (`[{ permission, scope }]`), so the clients build their menus from it (DL-52).
  - ⚠ NIK masking: show only the last 4 digits (`************1234`). Comment why (PII, persist `05-personnel-and-squads.md` AC-W23-6).
- **Table `devices`** (tenant table, RLS) — keep the 01.4 design. This is SiAPP's **push-token** registry; the one-phone binding is the separate `device_bindings` table (01.1).
  - `id`, `organization_id`, `personnel_id`, `session_id` (UNIQUE, `ON DELETE CASCADE`)
  - `platform` ENUM(`ANDROID`,`IOS`,`WEB`), `push_token` (UNIQUE within the organization), `app_version`, `last_seen_at`, timestamps
- **`POST /notifications/token`** (idempotent) upserts the device for the caller's live session. **`DELETE /notifications/token`** removes it.
  - ⚠ Keep the 01.4 rules and their comments: a device follows the login chain (refresh moves it to the new session); a push token belongs to one device (remove it from any other device first); a device never outlives its login.
- **People sync**
  - `identity.People.Ensure(ctx, nik)` (exported): returns the local `personnels.id` for a NIK, fetching from `personel.Directory` and inserting the row if needed. A NIK whose current company is not the caller's organization → `NOT_FOUND`. Module 02 uses it when it adds team members who have never signed in.
  - Periodic task `identity:resync-people` (daily, per organization): re-fetch people whose `synced_at` is older than 24 h in batches of 200 (`EmployeesByNIKs`). Mark people Personalia no longer returns, reports inactive, or reports under another company as `INACTIVE` and end their sessions. People are not matched against Personalia's devices (SiAPP has its own).
  - Comment that the Personalia webhook (05.3) makes this faster; the daily job is the safety net.

**Tests**
- `GET /auth/me` masks the NIK and never returns the device token.
- Device reset: an admin resets a bound phone → the binding is gone, the person's `MOBILE` sessions are revoked and their access tokens get 401, and the person can bind a new phone; a `PROJECT_ADMIN` cannot reset a person outside their projects; the reset is audited. HR's reset in Personalia does not affect SiAPP's binding.
- Admin gate: a person without an assignment gets 403 on `/admin/**` and cannot log in with `client = WEB`; a `PROJECT_ADMIN` sees only their projects; an admin cannot remove their own role; the seed command creates the first admin.
- Device: registering twice updates one row; refresh moves the device; a second person registering the same push token removes the first's; logout deletes it.
- `Ensure` creates a person once; a parallel call does not create two rows; a NIK from another company is not found.
- Resync: a person the fake deactivates becomes `INACTIVE` and their refresh fails.
- Cross-tenant: org B's token cannot read or delete org A's device.
