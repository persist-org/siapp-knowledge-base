# 00: Platform

**Purpose:** The foundation every module builds on: Postgres with tenant isolation, the contract and its envelopes, audit, idempotency, and the client for Persist Personel. No business features.
**Needs:** nothing.
**LOCK:** F1 (contract), F3 (tenant + RLS), and the client half of F2/F4/F5.
**Error reasons** (in `error.details.reason`; `error.code` is the contract enum):
- `ORG_CONTEXT_MISSING`
- `OCCURRED_AT_IN_FUTURE`, `OCCURRED_AT_TOO_OLD`
- `IDEMPOTENCY_KEY_REUSED` (code `CONFLICT`)
- `PERSONEL_UNAVAILABLE` (code `INTERNAL_ERROR`, HTTP 503)

---

## Phase 00.1: Project scaffold

**Goal:** A running Go API with config, structured logging, a health check, local infrastructure and a build pipeline, in the `pkg/` layout.

**Build**
- `go mod init`. Create `cmd/api/main.go` and add `import _ "time/tzdata"`.
  - ⚠ Comment why: it embeds the timezone database, so `time.LoadLocation("Asia/Jayapura")` works in minimal containers.
- `pkg/platform/config`: a `Config` struct parsed by `caarlos0/env`.
  - Variables: `APP_ENV`, `HTTP_ADDR`, `DATABASE_URL` (connects as `siapp_app`), `MIGRATE_DSN` (connects as the owner; read only by `cmd/migrate` and tests), `REDIS_URL`, `CORS_ORIGINS`, `LOG_LEVEL`.
  - Startup fails fast with a readable list of missing or invalid variables.
- Logging: a `slog` JSON handler.
- `pkg/platform/httpx`:
  - Router built with chi, with middleware for:
    - request ID (read `X-Request-Id`, or generate one and echo it back);
    - recoverer (log the stack, respond with `INTERNAL_ERROR`);
    - access log (method, path, status, duration, request ID, and **never** the Authorization header or the body);
    - CORS;
    - server timeouts (read 15 s, write 30 s, idle 60 s).
  - Graceful shutdown on SIGINT/SIGTERM.
- `GET /api/v1/health` → `{ "status": "ok" }`.
- Docker files, exactly as written in `docs/getting-started.md` Part 3 (if they already exist from that guide, commit them in this phase):
  - `docker-compose.yml`: `postgres:16`, `redis:7`, `bitnamilegacy/minio` (bucket `persist-media`), and the profiles `app` (migrate, api, worker) and `cctv` (MediaMTX). (00.1 also scaffolded a `stub` profile; Phase 00.5 removes it.)
  - `deploy/postgres/init.sql`: creates the app role `siapp_app` (`LOGIN NOSUPERUSER NOBYPASSRLS`). The owner `siapp_owner` is the container's `POSTGRES_USER`.
  - `Dockerfile`: multi-stage build on the `golang` image matching `go.mod` → `gcr.io/distroless/static-debian12:nonroot`, with `CGO_ENABLED=0`, building `./cmd/...` so every command (`/api`, `/worker`, `/migrate`, `/seed`) is in the image.
  - `.env.example`.
- `Makefile` targets: the Docker ones from the guide (`up`, `up-app`, `up-cctv`, `down`, `reset-db`, `logs`, `ps`, `psql`, `psql-app`) plus `run`, `worker`, `migrate`, `generate`, `lint`, `test`, `check`. Add `.golangci.yml` with the linters listed in `docs/tech-stack.md`.

**Tests**
- Unit: config parsing fails with a clear message when `DATABASE_URL` is missing.
- HTTP (`httptest`): health returns 200 and echoes the request ID.

---

## Phase 00.2: Postgres, tenant isolation, sqlc, migrations and test kit

**Goal:** Postgres is connected through sqlc/pgx, every tenant query is isolated by RLS, migrations run as the owner, and HTTP tests run against a real database as the app role.

**Build**
- `db/migrations/00001_00_2_init.sql`:
  - `CREATE EXTENSION IF NOT EXISTS btree_gist; CREATE EXTENSION IF NOT EXISTS pg_trgm;`
  - **Table `organizations`** (the tenant registry, **no RLS**): `id BIGINT PRIMARY KEY` (Personalia's company id, `m_companies.id`, not generated here; D11), `name`, `status` ENUM(`ACTIVE`,`SUSPENDED`), `first_seen_at`, `created_at`, `updated_at`. `siapp_app` gets `SELECT`, `INSERT`, `UPDATE` only.
  - ⚠ The block comment on conventions: BIGINT identity ids; `timestamptz` in UTC; the `organization_id` column default that reads `app.current_org`; the RLS policy template from the README; why the app role must not own the tables (owners and `BYPASSRLS` roles skip RLS, and `FORCE` closes the owner gap).
  - `ALTER DEFAULT PRIVILEGES` so tables the owner creates later are usable by `siapp_app`.
- `db/embed.go` embeds the migrations. `cmd/migrate` runs goose up/down/status with `MIGRATE_DSN`.
- `sqlc.yaml`: engine `postgresql`, `sql_package: pgx/v5`, schema `db/migrations`, queries `db/queries`, output `internal/db`, `emit_json_tags: false`, `emit_interface: false`, `emit_empty_slices: true`.
- `db/queries/platform.sql`: `-- name: Ping :one` (`SELECT 1`), and the `organizations` queries (`GetOrganization`, `UpsertOrganizationSeen`).
- `pkg/platform/database`:
  - `Open(ctx, url)` builds a `pgxpool`. ⚠ Set `AfterRelease` to run `DISCARD ALL` and use `QueryExecModeExec` for statements that set session state. Comment why: defence in depth, so a pooled connection can never carry one tenant's setting into another request (DL-56).
  - ⚠ **`WithTenant(ctx, pool, orgID, fn func(q *db.Queries) error) error`**: begins a transaction, runs `SELECT set_config('app.current_org', $1, true)` (transaction-local, parameterised), calls `fn`, commits on nil, rolls back on error or panic, then re-panics. Comment that this is the **only** way to read or write tenant data, and why the setting is transaction-local.
  - `WithoutTenant(ctx, pool, fn)`: a plain transaction, allowed only for the `organizations` registry. Comment the rule.
  - `IsUniqueViolation(err)` and `ConstraintName(err)` (SQLSTATE `23505`), `IsExclusionViolation(err)` (`23P01`), `IsRLSViolation(err)` (`42501`).
- Migrations run through `make migrate`, plus an optional run on API startup when `MIGRATE_ON_START=true` (dev only; it uses `MIGRATE_DSN`).
- Health check pings the database. It returns 503 when the database is down.
- `pkg/platform/testkit`:
  - `NewDatabase(m *testing.M)` for `TestMain`: creates `siapp_test_<pkg>_<rand>` as the owner, runs goose up, then hands tests a pool that connects as **`siapp_app`**. Drops the database on exit.
  - `Reset(t)`: `TRUNCATE … RESTART IDENTITY CASCADE` every table except `goose_db_version`. ⚠ Read the table list from `pg_tables`, so new tables are covered automatically.
  - `NewOrg(t)`: inserts an organization and returns its id; most tests create two.
  - `NewServer(t, deps)`: builds the real router with an `httptest.Server`.
  - In `-short` mode, skip with a message instead of failing.
- CI (`.github/workflows/ci.yml`): Postgres and Redis service containers (with the init script), install `sqlc`, `oapi-codegen` and `golangci-lint`, run `make check`.

**Tests**
- HTTP: health returns 200 with the database up.
- `WithTx`-style rollback: `WithTenant` rolls back when `fn` returns an error.
- ⚠ **Isolation tests** (these are the proof for LOCK F3, keep them forever):
  - A throwaway tenant table created in the test migration set: rows inserted under org A are invisible under org B (`SELECT` returns 0 rows).
  - Inserting a row with an explicit `organization_id` of another org fails with SQLSTATE `42501`.
  - Without `WithTenant`, a `SELECT` on a tenant table as `siapp_app` returns 0 rows.
  - The setting does not leak: run org A then org B on the same pooled connection (pool size 1) and check B sees only B.
  - **Guard test:** every table in schema `public` that has an `organization_id` column has `relrowsecurity` and `relforcerowsecurity` set and a policy named `org_isolation`. A table added later without RLS fails this test.

---

## Phase 00.3: Contract, envelopes and HTTP conventions

**Goal:** The vendored PERSIST contract, generated API types, the contract's envelopes and error shape, pagination, dates and clock. The shapes, status codes and SiAPP rules are specified in [`docs/api-response.md`](../api-response.md); build to it.

**Build**
- **Contract vendoring.**
  - `make contract` copies `persist-ecosystem/contracts/openapi.yaml` (path from `CONTRACT_SRC`, default `../persist-ecosystem/contracts/openapi.yaml`) to `api/openapi.yaml`, and runs `oapi-codegen` with `generate: models` into `internal/apigen`. `make check` fails if either differs from what is committed.
  - ⚠ Comment in `api/gen.go` why only models are generated: routes stay hand-mounted on chi so the default-deny middleware and per-route permissions stay explicit, while DTOs can never drift from the contract.
  - **Contract changes (PR to `persist-ecosystem`)** for decision D5: mark the operations SiAPP will not implement as Personel-owned (`x-owner: personel`) in their descriptions: `/admin/regions*`, `/admin/branches*`, `/admin/clients*`, `/admin/personnels*`, `/admin/positions*`, `/admin/web-users*`, `/auth/forgot-password`, `/auth/change-password`, `/platform/**`, `/billing/**`, `/developer/**`, `/admin/master-reviews*`, `/personnel-reviews*`.
- `pkg/platform/apperr`:
  - `type Error struct { Status int; Code apigen.ErrorCode; Reason, Message string; Details map[string]any; Fields []FieldError }`, which implements `error`.
  - Constructors `New(status, code, reason, msg)`, `WithDetails`, `WithFields`.
- `httpx.Decode(r, dst)`:
  - 1 MB `MaxBytesReader`, `DisallowUnknownFields`, then `validator.Struct`.
  - Validation errors become `VALIDATION_ERROR` with `errors: [{ field, code, message }]`, matching the contract's `FieldError`.
  - ⚠ Register a tag-name function so `field` shows the **JSON** name, not the Go field name.
- Responses:
  - `httpx.OK(w, r, status, data)` writes a `SuccessEnvelope` with `meta.request_id`.
  - `httpx.List(w, r, data, page)` writes a `ListEnvelope`.
  - `httpx.Delta(w, r, data, deleted, meta)` writes the delta-sync shape (DL-57).
  - `httpx.Error(w, r, err)`: maps `*apperr.Error` to its status and an `ErrorEnvelope` (with `details.reason`); maps everything else to 500 `INTERNAL_ERROR`, logging it with the request ID and never exposing it.
- Pagination: `httpx.Page{Page, Limit, Total}` and `httpx.ListQuery`, which parses `q`, `date`, `month`, `category_id`, `page` and `limit` from the URL query. `date` and `month` together → `VALIDATION_ERROR`. `limit` above 100 → `VALIDATION_ERROR`.
- `pkg/platform/timeutil`:
  - `Date` type (JSON `"YYYY-MM-DD"`; pgx scan/encode for `date`);
  - `BusinessDate(t time.Time, loc *time.Location) Date`;
  - `At(d Date, minuteOfDay int, loc) time.Time` (returns UTC);
  - `MonthRange("YYYY-MM") (first, last Date)`;
  - `SupportedTimezones` (`Asia/Jakarta`, `Asia/Makassar`, `Asia/Jayapura`).
- `pkg/platform/clock`: `Clock` interface with `System{}` and `Fixed{T}`.
- testkit: a client helper that validates each request and response against `api/openapi.yaml` (`openapi3filter`).

**Tests**
- Unit:
  - `BusinessDate` for 23:30 UTC in all three Indonesian zones;
  - `At` for `Asia/Jayapura`;
  - `MonthRange` for a leap-year February;
  - `Date` JSON and SQL round-trip.
- HTTP: a test-only route returns the `VALIDATION_ERROR` envelope with field errors, an unknown field is rejected, and both match the contract's `ErrorEnvelope`.

---

## Phase 00.4: Audit log and idempotency

**Goal:** Shared building blocks for sensitive writes and offline submissions.

**Build**
- **Table `audit_logs`** (tenant table, RLS, append-only):
  - Columns: `id`, `organization_id`, `actor_personnel_id` (nullable, **no FK**), `action` (e.g. `attendance.absence.create`), `entity_type`, `entity_id`, `metadata` (jsonb), `created_at`.
  - Partitioned by month on `created_at` (`PARTITION BY RANGE`), with the first 3 months created by the migration and later ones by a monthly asynq task `audit:partitions`.
  - Indexes: (`organization_id`, `entity_type`, `entity_id`, `created_at`) and (`organization_id`, `actor_personnel_id`, `created_at`).
  - `siapp_app` gets `INSERT` and `SELECT` only. ⚠ Comment why: no `UPDATE`/`DELETE` grant makes it append-only at the database level. Also comment why there is no FK: audit rows must outlive the rows they describe.
- **`audit.Record(ctx, q *db.Queries, Entry)`.** It takes the tenant transaction's `Queries`, so the audit row commits or rolls back together with the change. `metadata` must never hold a token, password, NIK or camera URL.
- **Table `idempotency_keys`** (tenant table, RLS):
  - `organization_id`, `actor_personnel_id`, `key` (uuid), `method`, `path`, `request_hash` (SHA-256 of the canonical body), `status_code`, `response_body` (jsonb), `created_at`.
  - Primary key (`organization_id`, `actor_personnel_id`, `key`). Index on `created_at` for cleanup.
- **`pkg/platform/idem`:**
  - ⚠ `Middleware` for the routes a phase marks idempotent:
    1. A missing `Idempotency-Key` on a route that requires it → `VALIDATION_ERROR`.
    2. Look up (actor, key). If found with the same `request_hash`, replay the stored status and body and set the header `Idempotent-Replayed: true`. If found with a different hash → 409 `CONFLICT` with reason `IDEMPOTENCY_KEY_REUSED`.
    3. Otherwise run the handler, then store the response **only for 2xx and 4xx other than 409/429**, so a transient failure can be retried.
    4. Two concurrent first requests with the same key: the insert of the key row uses `ON CONFLICT DO NOTHING`; the loser waits briefly and replays the winner's response, or gets 409 if it is still in flight.
  - Comment the replay contract: handlers check `idem.Replayed(ctx)` and skip side effects.
  - `OfflineCreate` struct for offline-capable request bodies: `LocalID` (uuid, must equal the header), `OccurredAt`. `CheckOccurredAt(occurredAt, now)`: more than 5 min in the future → `OCCURRED_AT_IN_FUTURE`; more than 7 days old → `OCCURRED_AT_TOO_OLD`.
  - Periodic task `idem:cleanup` (daily, per organization): delete keys older than `idempotencyKeyTTL = 8 * 24h` (longer than the 7-day offline window).

**Tests**
- Unit: `CheckOccurredAt` boundaries; request hash is stable across key order in JSON.
- HTTP with a test route: first call runs; a replay returns the same body and does not run the handler; a different body with the same key returns 409; a 500 is not stored; two parallel first calls run the handler once.
- Cross-tenant: the same key from another organization is a different key.
- Audit: an `UPDATE` on `audit_logs` as `siapp_app` fails.

---

## Phase 00.5: Personalia accounts (real integration) 🔌

**Goal:** People log in to SiAPP with their **real Personalia account**. That needs three things:
- `kpi-personalia-api` serves the integration endpoints for real: login and employee lookup.
- A new `m_users.domain` column says which products an account may use.
- SiAPP gets the one client package that every module uses to reach Personalia.

There is no standalone stub. Local development runs against the real `kpi-personalia-api`. SiAPP's automated tests use a small in-process fake with the same JSON, so CI needs no MySQL and outages can be simulated.

**Decisions taken here**
- **`m_users.domain`**, `ENUM('personalia','siapp') NOT NULL DEFAULT 'personalia'`.
  - `personalia`: the Personalia app, the Personalia admin panel and SiAPP.
  - `siapp`: SiAPP only. Refused by the Personalia app login and by the admin panel login.
  - Existing accounts default to `personalia`, so nobody loses access.
- **Who sets it:** HR, in the admin panel's user form.
- **Where the migration lives:** a new migration system in `kpi-personalia-api` (goose, MySQL dialect), not a Laravel migration.
- **Phone binding is SiAPP's own** (Phase 01.x).
  - The integration login never reads or writes `t_employee_device`, so the Personalia app and SiAPP never block each other on one phone.
  - HR's "reset device" in the admin panel stays Personalia-only.
  - This replaces the device steps in `personalia-integration.md` P.1.
- **Seams in this phase:** login (S1) and employee lookup (S4) are built for real. Org (S2) and attendance (S3) get their client interfaces and fake responses only. Their endpoints come in the phase of the SiAPP module that first needs them (`personalia-integration.md`, "Later seams").

### Part A: `kpi-personalia-api` (migrations, domain, integration endpoints)

Follow that repo's conventions (`internal/modules/<name>/{handler,service,dto,errors}.go`, sqlc queries in `db/query/`, config in `configs/config.example.yaml`), not SiAPP's.

**Build**
- **Migration system** (new in this repo):
  - `db/migrations/NNNNN_<desc>.sql` (goose, `-- +goose Up` and `-- +goose Down`), embedded, run by `cmd/migrate up|down|status` and `make migrate`.
  - It connects with `database.migrate_dsn`, an account allowed to `ALTER`. When that is empty it falls back to `database.dsn`.
  - ⚠ Write the coexistence rules in the README and as a comment in the first migration:
    - The database is shared with the Laravel admin panel, which keeps its own `migrations` table. Goose keeps `goose_db_version`.
    - Laravel stays the owner of the tables it created. Go migrations only *add* (columns, indexes, new tables).
    - Never run `artisan migrate:fresh` or `migrate:rollback` against a shared database: they know nothing of goose's changes.
  - The schema snapshot `db/schema/kpi_personalia.sql` (the sqlc input) is updated in the same change as each migration. A test fails if a column a migration adds is missing from the snapshot.
- **Migration `00001_users_domain.sql`:**
  ```sql
  ALTER TABLE m_users
    ADD COLUMN domain ENUM('personalia','siapp') NOT NULL DEFAULT 'personalia' AFTER role_id;
  ```
  Down drops the column.
- **`domain.User.Domain`** (`personalia` | `siapp`), read by every user query that builds a `domain.User`.
- **The Personalia app refuses `siapp` accounts:**
  - `auth.Service.Login`: after the credentials are verified, a `siapp` account → 403 with a new error `ErrDomainNotAllowed` (message key `auth.domain_not_allowed`: "Akun ini hanya dapat digunakan di aplikasi SiAPP.").
  - ⚠ The check runs **after** the password check. Comment why: answering "wrong app" before the password would tell a guesser that the email exists.
  - `auth.Service.Refresh` and the token check (`AuthSessionState`) also refuse a `siapp` account. Then HR switching an account to `siapp` ends its open Personalia sessions, the same way deactivation does.
- **Integration key and router** (from `personalia-integration.md` P.1, unchanged):
  - The config section `integrations.siapp.key_hashes` and `login_failures_per_email`.
  - `middleware.RequireIntegrationKey`.
  - The `/api/v3/integrations/siapp` group, outside `Authenticate` / `VerifyDevice`.
- **`auth.Service.VerifyCredentials(ctx, email, password) (domain.User, domain.EmployeeWork, error)`**, extracted from `Login`: the email lookup, the dummy-hash compare, bcrypt, the latest work row and `IsEmployeeInactive`. Both logins use it, so they cannot drift.
- **`POST /api/v3/integrations/siapp/login`** `{ email, password, user_agent?, client_ip? }`:
  1. `VerifyCredentials`. A wrong email or password → 401 `invalid_credentials`, one answer for both.
  2. No work row → 403 `employee_work_missing`; inactive → 403 `employee_inactive`.
  3. Return `Identity`. Both domains are accepted: SiAPP is open to `personalia` and `siapp` accounts alike.
  - **No device handling at all.**
  - Issues no token, revokes nothing, and touches neither `t_employee_device` nor FCM tokens.
  - The per-email failure limit returns 429.
  - `client_ip` and `user_agent` go in the security log. Passwords are never logged.
- **`POST /api/v3/integrations/siapp/employees`** `{ niks: [...] }` (1–200) → `{ data: [Identity...] }`, as in P.2: one batched query; unknown NIKs omitted; inactive people returned with `active: false`.
- **`Identity`** (this replaces the shape in `personalia-integration.md`):
  ```json
  {
    "user_id": 123,
    "email": "budi@contoh.com",
    "nik": "0012345678",          // null when the account has no NIK (D4)
    "full_name": "Budi Santoso",
    "phone": "0812...",          // null when unknown
    "photo_url": "https://...",  // null when unknown
    "company": { "id": 7, "name": "PT ..." },
    "client":  { "id": 41, "name": "PT ..." },
    "position": { "id": 3, "name": "Satpam" },
    "active": true,
    "domain": "siapp"             // "personalia" | "siapp"
  }
  ```
  `device_token_sha256` is gone: SiAPP binds phones itself.
- **Dev seed** `db/seed/40_siapp.sql` (idempotent), all with one dev password printed by `make seed`:
  - two companies;
  - per company, a DANRU and two members (`siapp` domain);
  - one `personalia` account that also uses SiAPP;
  - someone without a NIK;
  - someone deactivated;
  - someone placed at a client different from their company.

**Tests**
- Migrations: up, down, up on a scratch MySQL. `domain` defaults to `personalia` for existing rows. The snapshot contains the column.
- Personalia app login:
  - A `siapp` account with the right password → 403 `domain_not_allowed`; with a wrong password → the usual 401.
  - A `personalia` account logs in as before.
  - Switching an account to `siapp` makes its next refresh fail.
- Integration login:
  - No key or a wrong key → 401; a rotated second key works; empty `key_hashes` → 404.
  - Correct password → `Identity` with `domain`, for both domains.
  - A wrong password and an unknown email → the same 401. Resigned → 403. No work row → 403.
  - `t_employee_device` and `t_refresh_tokens` are unchanged after any integration login.
  - The 11th failure for one email within 15 minutes → 429, whatever the `client_ip`.
- Employees: known, unknown and inactive NIKs in one batch; 201 NIKs → 422; a person moved to another company returns the new `company.id`.
- The existing `auth.Service.Login` and `Refresh` tests still pass after the extraction.

### Part B: Personalia admin panel (`PJ48-KPI-Personalia-AdminPanel`, Laravel)

**Build**
- `App\User`: `domain` in `$fillable`, constants `DOMAIN_PERSONALIA` and `DOMAIN_SIAPP`.
- **User form** (create and edit, wherever the panel creates `m_users` rows: the user management screen and the employee screens that create a login):
  - a "Akses aplikasi" select (`Personalia + SiAPP` / `SiAPP saja`), default `personalia`, validated `in:personalia,siapp`;
  - the user list shows the value.
- **Login gate:** the web login (`AuthController@doLogin`) and the legacy mobile API login (`Api/V1/AuthController`, while it is still served) refuse a `siapp` account **after** the password check, with the same message as the Go API. A `siapp` account that is already logged in is logged out on its next request (middleware on the authenticated group).
- No migration here: the column comes from Part A.

**Tests** (in the panel's own test setup, or a manual checklist if it has none)
- HR creates a `siapp` user; it cannot log in to the panel, but its credentials work through the integration login.
- Switching an existing user to `siapp` logs them out of the panel.

### Part C: SiAPP (`pkg/platform/personel`)

**Build**
- **`pkg/platform/personel`** (the only code that calls Personalia). Interfaces, each with a doc comment naming its seam (S1–S6 in the README):
  - **`Identity`**: `PersonelUserID, Email, NIK (may be empty, D4), FullName, Phone, PhotoURL, OrganizationID, OrganizationName (the employing company, D11), ClientID, ClientName, PositionID, PositionName, Active, Domain`. `Domain` is `DomainPersonalia` or `DomainSiAPP`.
  - **`Auth.Login(ctx, LoginInput{ Email, Password, ClientIP, UserAgent }) (Identity, error)`** (S1). Typed errors:
    - `ErrInvalidCredentials` (401);
    - `ErrAccountDisabled` (403 `employee_inactive` or `employee_work_missing`);
    - `ErrRateLimited` (429);
    - `ErrUnavailable` (outage, timeout, open breaker → `PERSONEL_UNAVAILABLE`).
    - There is no device error: binding is SiAPP's (01.x).
  - **`Directory`**: `EmployeeByNIK`, `EmployeesByNIKs` (S4).
  - **`Org`**: `Projects`, `Project(id)`, `Placements(projectID)`, `Positions` (S2, proposed).
  - **`Attendance`**: `PushEvents`, `Status` (S3, proposed).
- **HTTP implementation `personel.Client`:**
  - `PERSONEL_BASE_URL`, e.g. `http://localhost:8081/api/v3/integrations/siapp` in development.
  - `PERSONEL_INTEGRATION_KEY`, sent as `X-Integration-Key`.
  - `personelTimeout = 5 * time.Second`.
  - It decodes `kpi-personalia-api`'s envelope (`data`, `error`, `message`, `internal_code`, `error_type`) and maps `internal_code` to the typed errors.
  - ⚠ Comment why a shared key and not OAuth (D8).
  - Retries idempotent calls (3 tries with backoff). **Never** retries `Auth.Login`: a retry is a second failed attempt against the account.
  - ⚠ The circuit breaker opens after 5 consecutive failures for 30 s. 4xx answers are not failures.
  - A 401 on the integration key is a configuration error: logged loudly, and returned as `ErrUnavailable`, never as "invalid credentials".
  - Never logs the key, passwords, NIK or response bodies.
- **Cache `personel.Cached`:**
  - Redis read-through for `Org` and `Directory`, keys `org:{orgID}:personel:{kind}:{id}`, `personelCacheTTL = 10 * time.Minute`.
  - `Invalidate(orgID, kind, id)` for the webhook receiver (05.3).
  - `Auth` is never cached.
- **Fake `pkg/platform/personel/personeltest`:**
  - An in-process `http.Handler` with the same paths, envelope, status codes and `internal_code`s as Part A, behind the same key check.
  - Fixtures mirror Part A's dev seed.
  - Hooks for tests:
    - `Deactivate(nik)`, `SetDomain(email, domain)`, `MoveToCompany(nik, id)`;
    - `FailNext(status)`, `Down()` / `Up()`;
    - `Calls()`, to assert that login was not retried.
  - Records pushes for the proposed seams.
  - `testkit` serves it in-process, and `NewServer` wires the client to it. It is the only fake. Later phases' "against the stub" means this fake.
- **A contract test against the real API**, `go test -tags personel_live ./pkg/platform/personel`.
  - It runs only when `PERSONEL_LIVE_URL` and `PERSONEL_LIVE_KEY` are set, and skips otherwise.
  - It logs in two seeded people and looks up their NIKs. It fails if the real responses decode differently from the fake's.
  - Run it whenever either side changes.
- **Remove the standalone stub** that 00.1 scaffolded:
  - the `personel-stub` service, the `stub` profile in `docker-compose.yml` and `make up-stub`;
  - the `cmd/personel-stub` mention in the Dockerfile comment.
  - `docs/dev-setup.md` and `.env.example` point `PERSONEL_BASE_URL` at a local `kpi-personalia-api`, and explain how to put the SHA-256 of the dev key in its `integrations.siapp.key_hashes`.
- **Contract (PR to `persist-ecosystem`):**
  - Add a `Seam` tag with S1–S6.
  - S1 login and S4 employees are marked `x-status: implemented`; the others `x-status: proposed`, so the Personel team can review them (LOCK F1).

**Tests** (against the fake)
- `Auth.Login` returns the identity, with `Domain`, for a `personalia` account and for a `siapp` account.
- `Auth.Login` maps 401, 403 (both reasons) and 429 to the typed errors, and is never retried (`Calls()` shows one request).
- A wrong integration key fails as a configuration error, not as invalid credentials.
- `Directory` retries a 503 and then succeeds; `Auth` does not.
- The circuit breaker opens and recovers with a fixed clock.
- The cache serves the second call without hitting the fake; `Invalidate` forces a refetch.
- With the live tag: the contract test above.

**Knock-on changes for later phases** (update their plan files before building them)
- **01-auth:** the phone binding moves into SiAPP.
  - `AUTH_DEVICE_BOUND` is decided by SiAPP's own binding, not by a Personalia `409`.
  - `AUTH_DEVICE_CHANGED` comes from SiAPP's binding, not from a `device_token_sha256` change.
  - SiAPP needs its own "reset device" for admins, because HR's reset in Personalia no longer covers SiAPP.
  - The tests "the stub rebinds the person to another device" become SiAPP-side tests.
- **personalia-integration.md:** P.1 and P.2 are delivered by this phase. Its `Identity` and device steps are replaced by the ones above.
- **README:** the "stub" wording (steps marked 🔌, the `cmd/personel-stub` line in the layout, the testkit line) refers to the in-process fake.
