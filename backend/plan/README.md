# SiAPP Backend: Implementation Plan

This plan builds the backend of the **PERSIST Security add-on**: the rebuild of SiAPP, serving the `security` and `cleaning` business lines from one codebase. It runs **on top of Persist Personel**, the base platform.

**Sources, in order of authority:**
1. `persist-ecosystem/docs/LOCK/` (locked September 2026). It wins any conflict.
2. `persist-ecosystem/contracts/openapi.yaml`: the API contract.
3. The other `persist-ecosystem/docs/` files (domain, data model, mobile, web, decision log).
4. The SiAPP live app, where the manual is out of date (`persist-ecosystem/docs/06-traceability/05-siapp-live-discovery.md`).
5. `source/manual-book.txt` (SiAPP User Manual v1.2, 2020): the field workflows.

**Related docs:**
- Why modules come in this order: `docs/backend-modules.md`.
- Stack and the reasons for it: `docs/tech-stack.md`.

This README holds the rules **every phase** follows. Each module file holds that module's phases only.

**Before Phase 00.1**, follow `docs/getting-started.md`: it installs the tools, starts the stack in Docker, archives the code built for the first plan and creates the rebuild branch.

---

## How to execute this plan

- **One phase per session.** Each phase is sized as one focused unit of work, roughly 5–12 files.
- **Prompt to use:**
  > Implement Phase `07.2` from `docs/plan/07-attendance.md`. Follow all conventions in `docs/plan/README.md`. Do not implement anything from other phases.
- **Before coding, read:**
  - this README;
  - the module file;
  - the code produced by the phases listed under **Needs**;
  - the contract operations the phase lists.
- **After coding:**
  - run `make check`;
  - commit as `feat(<module>): phase <id> <short title>`.
- **Order.** Execute phases in file order. Module files are numbered by priority. **One exception: the SiAPP → Personel sync comes last** (see "Personel sync is built last" below): its phases run after the last module, whatever their number.
- **Steps marked ⚠** contain non-obvious logic. They **must** get an explanatory comment in code: why it works this way, which edge cases it covers, and which manual section or decision (DL-xx, LOCK) it implements.
- **Steps marked 🔌** depend on a Persist Personel seam endpoint. Build them against the Personel client interface (`pkg/platform/personel`, Phase 00.5) and its in-process fake `personeltest`. Wherever a phase file says "the stub", it means that fake. Login (S1) and employee lookup (S4) already run against the real `kpi-personalia-api` in development; the other seams switch to the real Personel only when the seam is agreed in the contract.

---

## Ownership: what SiAPP builds and what it consumes

From LOCK/01 §2 ("one piece of data, one owner"). SiAPP **never** queries Personel's database. It goes through the seam (server-to-server API with an integration key + NIK) only.

| Domain | Owner | What SiAPP does |
|---|---|---|
| Identity, login, passwords, password reset, the one bound phone per person | **Personel** | Passes email + password to Personalia's integration login and issues its own SiAPP tokens (module 01). Stores no credentials. |
| Tenants (organizations) and SaaS billing | **Personel** | Takes `organization_id` (the person's employing company) from Personalia's login answer. Keeps a small local registry of the tenants it has seen. |
| Org hierarchy: company, division, department, position, level, project/area/location, placement (`m_pmds`) | **Personel** | Reads through the seam, with a cache (module 02). Keeps per-project **security settings** locally. |
| Employee profile (biodata, NIK) | **Personel** | Caches the fields it displays, keyed by NIK (module 01). |
| Attendance (system of record) | **Personel** | Captures QR and selfie check-ins with evidence, then pushes them to Personel (module 07). |
| Leave, permit and sick requests (izin, cuti, sakit), their categories, quotas and balances | **SiAPP** | Owned in full (module 07.5): the DANRU decides, an approved leave is a `LEAVE` attendance row. It copies how Personalia does it, and tells Personel's official attendance record through the outbox (D21). |
| Teams (regu), team roles (DANRU/MEMBER), shifts, posts, swaps (tukar jaga) | **SiAPP** | The operational overlay (LOCK/03 §1). Open decision D1 may move shifts to Personel. |
| Patrol and guard posts, panic, CCTV, tracking, reports, instructions, assets, notifications | **SiAPP** | Owned in full. |
| Cleaning features (E-Memo, Generate QR) | **SiAPP** | Same platform, behind `business_line` flags (LOCK/03 §2). |
| Performance reviews (Penilaian) | Performance add-on (deferred) | Not built here (LOCK/03 §1). |
| Payroll | Payroll add-on (deferred) | Not built here. |

---

## The Personel seam (LOCK F1)

The LOCK names four seams. The ownership table above needs a few more. **None of them exists yet.** Phase 00.5 defines the Go client interfaces and the in-process fake. We build the real endpoints ourselves in Personalia's Go API (`kpi-personalia-api`), under `/api/v3/integrations/siapp`, authenticated by a shared integration key: see [`personalia-integration.md`](personalia-integration.md) (D10).

| Seam | Proposed operations (to be agreed) | Used by |
|---|---|---|
| S1 Login | `POST /integrations/siapp/login`: email + password (+ device token) → identity, no Personalia token issued (P.1) | 01 |
| S2 Org read | List and get: companies, divisions, departments, positions/levels, projects/areas/locations, placements (`m_pmds`) for a location | 02, 03 |
| S3 Attendance push | `POST` attendance events (idempotent by event id); `GET` attendance status for a person and date range | 07, 14 |
| S4 Profile / NIK | `POST /integrations/siapp/employees`: batch of NIKs → identities (P.2) | 01, 02 |
| S5 Leave | **Retired.** Leave is SiAPP's (07.5). The one thing Personel still needs, an approved leave on its official attendance record, travels as an outbox event (07.5 C4, D21). | — |
| S6 Change events | Webhooks from Personel: employee updated or deactivated, placement changed, position changed | 02, 05 |

---

## Decisions (change here if wrong)

| Topic | Decision |
|---|---|
| Stack | Go + PostgreSQL 16 + sqlc/pgx (see `docs/tech-stack.md`). |
| Tenancy | **Multi-tenant.** Tenant = the person's employing company in Personalia (`organization_id` = `m_companies.id` from `m_employee_work.company_id`, D11). The placement client is not the tenant. It comes from Personalia's login answer and then the SiAPP token, **never** from the request. Postgres RLS enforces it. |
| Business lines | `projects.business_line` ∈ {`security`, `cleaning`}. Cleaning differs only by feature flags and UI labels. The backend never forks a module for it. |
| People | One local table `personnels`: a cache of Personel people keyed by `(organization_id, nik)`, with **no credentials**. Field staff and web admins are both rows here. |
| Field roles | `team_members.role` ∈ {`DANRU`, `MEMBER`}, per team membership (DL-04). A person can lead team A and be a member of team B. |
| Web roles | Local role assignment per person: `ORG_ADMIN` (whole organization) or `PROJECT_ADMIN` (listed projects) (Phase 01.3). |
| Language | Code identifiers are **English** (DL-30), using the contract's names. User-facing messages, notification text and seed data are **Indonesian**. |
| Time zones | Each project has an IANA timezone (`Asia/Jakarta`, `Asia/Makassar` or `Asia/Jayapura`) in its security settings. Timestamps are stored in UTC (`timestamptz`). Business dates are computed in the project timezone (DL-03). |
| Offline | Reports, patrol/guard scans, zone check-ins, the panic trigger and its follow-up report, and asset reports can be queued offline and replayed with `Idempotency-Key`. (The persist docs disagree on whether panic queues offline; this plan allows it, because a panic pressed without signal must still reach the team when signal returns, and the replay rule keeps it from sounding twice.) Attendance check-in is online-only, because QR tokens expire in 60 s. The push of attendance to Personel is asynchronous (outbox), so a Personel outage never blocks a check-in. |
| Contract gaps | Where the manual needs something the contract lacks (for example instruction feedback), the phase adds it to `persist-ecosystem/contracts/openapi.yaml` **through a PR to that repo** in the same phase, and lists it under "Contract changes". |

### Personel sync is built last

**Decision:** everything that SiAPP **sends to Personel** has the lowest priority in the whole plan. SiAPP's own features are built first, and every one of them already works on its own: the Personel side is the in-process fake (`personeltest`, Phase 00.5) until the sync is built.

| Built last (after module 16) | What it is |
|---|---|
| **07.3** Delivery to Personel | The outbox worker that pushes attendance events (S3), its admin resync, and its metrics. |
| **Switching S3 from the fake to the real Personel** | Needs the seam agreed in the contract and built (D10). |

**What is NOT delayed, and why:**
- Everything SiAPP captures and stores locally: attendance rows, the `personel_outbox` rows (attendance and leave events accumulate as `PENDING`, so nothing is lost and the worker can drain them later), leave requests with their quota and ledger, the DANRU's queue, the `LEAVE` status in the calendar, notifications.
- What Personel sends *to* SiAPP: login (S1), employee lookup (S4), org read (S2) and change events (S6). They are not "SiAPP → Personel sync", and login and org data are needed by everything else.

**What this costs, so nobody is surprised:** until the sync is built, **no attendance and no approved leave reaches Personel**: its official record misses them, so in the meantime they exist only in SiAPP. Nothing may go to production as the system of record before this is done.

### Open decisions

Each has a default so work can continue. Confirm before the phase that names it.

| # | Question | Default used by this plan | Blocks |
|---|---|---|---|
| D1 | Regu jaga: a new grouping, or reuse Personel's `m_harikerja` shifts + `t_location_employee`? (LOCK/04 §5) | New grouping in SiAPP (`teams`, `team_members`, `shift_templates`, `team_shifts`), behind interfaces that a Personel-backed version can replace. | 02.2, 02.3 |
| D2 | Who approves leave, and where? The manual has the DANRU approve in the app. | **Decided:** the DANRU approves in SiAPP, and SiAPP owns the whole leave flow (categories, quota, ledger, decision). Nothing is forwarded to Personel as a request; only the approved days reach its attendance record, see D21. An admin can stand in or overrule. | 07.5 |
| D3 | How do people sign in? | **Decided:** in SiAPP's own login screen with their Personalia email and password. `POST /auth/login` checks them through Personalia's integration login (S1); no Personalia token is involved (D8). | 01.1 |
| D4 | Do web admins without a NIK exist in Personel? | Key people by Personel user id; `nik` is nullable for non-employee accounts. | 01.3 |
| D5 | The contract still has local CRUD for regions, branches, clients, personnels, positions, web users, platform and billing. Under the LOCK these belong to Personel. **Leave categories are no longer in this list: SiAPP owns them (07.5).** | SiAPP does **not** implement them. A contract PR marks them as Personel-owned. | 00.3 |
| D6 | "Tagihan & langganan" appears under Security in LOCK/03 §1 but billing belongs to Personel in LOCK/01 §2. | Not built here until clarified. | — |
| D7 | CCTV vendor and protocol. | MediaMTX gateway (RTSP in, HLS/RTSP out). | 13.2 |
| D21 | Leave is SiAPP's now. How does an approved leave reach Personel's official attendance (so the day is not counted absent there)? | SiAPP writes `leave.approved` and `leave.revoked` events to `personel_outbox` (07.5 C4) and 07.3 delivers them with the attendance events (S3). The event shape is proposed to the Personel team (D10). Until agreed, the rows wait as `PENDING`. | 07.3 |

---

## Repository layout

```
cmd/
  api/main.go             HTTP server: wiring only
  worker/main.go          asynq worker + periodic scheduler: wiring only
  migrate/main.go         goose runner (up/down/status), runs as the owner role
  seed/main.go            dev seed (masters and a demo organization)
api/
  openapi.yaml            vendored copy of persist-ecosystem/contracts/openapi.yaml (do not edit here)
  gen.go                  oapi-codegen directive: types only
db/
  migrations/             goose SQL files: 00001_<phase>_<desc>.sql
  queries/<domain>.sql    sqlc queries, one file per domain
  embed.go                //go:embed migrations/*.sql
sqlc.yaml
internal/
  db/                     sqlc-generated code (never edited by hand)
  apigen/                 oapi-codegen output (never edited by hand)
pkg/
  platform/               cross-cutting helpers; NO business rules
    config/ database/ httpx/ apperr/ clock/ timeutil/ geo/ idem/ audit/ testkit/ personel/
  <domain>/               e.g. attendance/
    handler.go            HTTP: decode → call usecase → respond (thin); also mounts routes
    usecase.go            business rules
    repo.go               persistence: wraps the sqlc queries of this domain; runs inside WithTenant
    item.go               domain types, enums and mapping to/from the generated API types
    errors.go             the domain's error reasons (listed in its plan file)
    constants.go          named limits, TTLs, windows
    *_test.go             unit tests (pure logic) + HTTP tests (via testkit)
```

`internal/` holds only generated code. Everything hand-written lives under `pkg/` (LOCK/02 § Stack).

## Conventions

### Go code
- **Layers.** `handler` → `usecase` → `repo`. Handlers only decode, validate, call a usecase and write the response. All rules live in **usecases**. Only `repo` touches sqlc.
- **Tenant data path.** ⚠ A repo method receives a `*db.Queries` that `database.WithTenant` created. Nothing else may open a transaction on tenant tables. Side effects (notifications, push, realtime, outbox delivery) run **after** the commit.
- **Cross-domain dependencies** are small interfaces declared **in the consuming package**, for example `type notifier interface { Notify(ctx, notifications.Input) }`. `cmd/api` wires the real implementations. This avoids import cycles and makes fakes trivial in tests.
- **Query ownership.** A domain calls only the queries in its own `db/queries/<domain>.sql`. Anything it needs from another domain goes through that domain's exported usecase: `organization.Query`, `access.Service`, `media.Service`, and so on.
- **Personel access.** Only `pkg/platform/personel` talks to Personel. Domains depend on its interfaces.
- **Context.** `context.Context` is the first parameter of anything that does I/O. Never store a context in a struct.
- **Errors.** Wrap with `fmt.Errorf("verb noun: %w", err)`. Return `*apperr.Error` for expected failures. Never `panic` in request paths.
- **Time.** Inject `clock.Clock` into usecases and never call `time.Now()` directly, so tests can control time.
- **Constants.** Magic numbers go in `constants.go` with the unit in the name, e.g. `qrTokenTTL = 60 * time.Second`.
- **Comments.**
  - Explain *why*.
  - Every **⚠** item gets a block comment.
  - Every exported function or type gets a doc comment (Go convention).
  - No comments that restate the code.

### API (follows `contracts/openapi.yaml`)
The full response reference, with examples, the status ↔ code table and the auth failures, is [`docs/api-response.md`](../api-response.md).
- **Paths.** Base path `/api/v1`. Use the contract's paths and operation names exactly. Admin endpoints live under `/api/v1/admin/**`.
- **Types.** Request and response structs come from `internal/apigen` (oapi-codegen). Do not hand-write DTOs that the contract already defines.
- **JSON.**
  - Keys are snake_case.
  - Timestamps are UTC ISO-8601 with `Z` (DL-38).
  - Business dates are `YYYY-MM-DD` (the `timeutil.Date` type).
  - Money is a decimal string (DL-38).
- **Envelopes.**
  - A single resource: `SuccessEnvelope` → `{ success: true, data, meta: { request_id } }`.
  - A list: `ListEnvelope` → `{ success: true, data: [...], pagination: { page, limit, total, total_pages }, meta }`.
  - A delta-sync response: `SuccessEnvelope` + `deleted[]` + `DeltaMeta` (`server_time`, `next_cursor`, `has_more`), with no totals (DL-57).
  - An error: `ErrorEnvelope` → `{ success: false, error: { code, message, details?, errors?[{ field, code, message }] }, meta }`.
- **Error codes.** ⚠ `error.code` is always a value of the contract's `ErrorCode` enum (`VALIDATION_ERROR`, `UNAUTHENTICATED`, `TOKEN_EXPIRED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `DUPLICATE`, `UNPROCESSABLE`, `RATE_LIMITED`, `INTERNAL_ERROR`, …). The domain-specific reason that each module file lists (for example `ATTENDANCE_QR_EXPIRED`) goes in `error.details.reason`. `message` is Indonesian and safe to show to users.
- **Pagination.** `page` (default 1) and `limit` (default 20, max 100). Lists that would need an expensive `COUNT(*)` on a hot table use the delta-sync shape instead; each phase says which.
- **Shared list query.** `httpx.ListQuery` parses `q`, `date`, `month`, `category_id`, `page` and `limit`.
- **Idempotency.** ⚠ Every unsafe operation the contract marks with `Idempotency-Key` (DL-34) goes through `idem.Middleware`: same key and same body → the stored response, with no side effects; same key and a different body → 409 `CONFLICT`. Offline-capable creates also store the key in a `local_id` column with a UNIQUE index, so a replay after the key store expires still cannot create a duplicate.
- **Authentication is default-deny.**
  - `auth.Require` is mounted **once**, on the whole API. Every request needs a valid SiAPP Bearer JWT unless its method and path are on the explicit public allowlist (`auth.PublicRoutes`).
  - The public allowlist is exactly: `GET /api/v1/health`, `POST /api/v1/auth/login`, `POST /api/v1/auth/refresh`, the Personel webhook receiver (authenticated by signature instead, Phase 05.3), and the local storage's blob routes `PUT` and `GET /api/v1/media/blobs/{token}` (authorized by the signed token in the URL, Phase 04.1; with the `minio` and `s3` drivers they answer 404). A `{name}` segment in an allowlist path matches exactly one segment, never a prefix.
  - Adding a public route needs an explicit line in the allowlist **and** `security: []` in the contract. A test checks that the two agree.
- **Authorization.**
  - Field routes: `access.Require("<permission>")` middleware + scope checks in usecases (from Phase 03.2).
  - Admin routes: `access.RequireAdmin`, mounted on the `/api/v1/admin` group (Phase 01.3).
- **Contract.** Every endpoint a phase builds must exist in the contract. HTTP tests fail if a request or response does not match it.

### Data (PostgreSQL 16 + sqlc/pgx)
- **Tables.** snake_case, plural, contract names (`teams`, `team_members`, `personnels`, `attendances`, `patrol_zones`, `patrol_scans`, `panics`, `cctvs`, `location_logs`, …).
- **IDs.** `id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY`.
- **Tenant column.** ⚠ Every tenant table has `organization_id BIGINT NOT NULL DEFAULT current_setting('app.current_org')::bigint REFERENCES organizations(id)`, and every index on it leads with `organization_id`. Comment once, in the first migration, why the default reads the session setting: inserts cannot forget the tenant, and RLS `WITH CHECK` rejects a mismatch.
- **RLS.** Every tenant table gets, in its own migration:
  ```sql
  ALTER TABLE x ENABLE ROW LEVEL SECURITY;
  ALTER TABLE x FORCE ROW LEVEL SECURITY;
  CREATE POLICY org_isolation ON x
    USING (organization_id = NULLIF(current_setting('app.current_org', true), '')::bigint)
    WITH CHECK (organization_id = NULLIF(current_setting('app.current_org', true), '')::bigint);
  GRANT SELECT, INSERT, UPDATE, DELETE ON x TO siapp_app;
  ```
  A test (Phase 00.2) fails if any table in `public` that has an `organization_id` column lacks `FORCE` or the policy.
- **Timestamps.** `timestamptz`, always UTC. Every table has `created_at` and `updated_at`. `deleted_at` only where a phase says so; queries filter `deleted_at IS NULL` explicitly.
- **Business dates** (`work_date`, `occurred_date`) are `date` columns computed in the project timezone. Never filter by day on a UTC timestamp.
- **Enums** are Postgres `CREATE TYPE … AS ENUM`, which sqlc turns into typed Go constants. Flexible payloads are `jsonb`.
- **Uniqueness among live rows** uses a partial unique index: `CREATE UNIQUE INDEX … ON teams (organization_id, project_id, name) WHERE deleted_at IS NULL`.
- **Date-range overlaps** use an exclusion constraint where the rule fits one table: `EXCLUDE USING gist (organization_id WITH =, team_id WITH =, personnel_id WITH =, daterange(start_date, end_date, '[]') WITH &&)` (needs `btree_gist`).
- **Optional filters** in sqlc: `(sqlc.narg('category_id')::bigint IS NULL OR category_id = sqlc.narg('category_id'))`.
- **Scope lists** use `id = ANY(sqlc.arg('team_ids')::bigint[])`. Return early in Go when the slice is empty; do not run the query.
- **Upserts** use `INSERT … ON CONFLICT (…) DO UPDATE SET … WHERE …`. The `WHERE` on `DO UPDATE` is how a phase skips rows that must not change.
- **Batch inserts** use sqlc `:copyfrom`.
- **Search** uses `pg_trgm` GIN indexes with `ILIKE '%' || q || '%'`. Escape `%` and `_` in user input.
- **Concurrency.** Use `SELECT … FOR UPDATE` inside the transaction wherever two requests could race. Each phase names these places.
- **Migrations.**
  - One goose file per phase: `NNNNN_<phase>_<desc>.sql`, with both `-- +goose Up` and `-- +goose Down`.
  - Never edit a migration once it has been applied anywhere shared.
- **Cross-tenant work.** Nothing bypasses RLS: the app role has no `BYPASSRLS` and no `SECURITY DEFINER` functions are needed. Background jobs either carry `organization_id`, or loop over `organizations` and run each organization in its own `WithTenant`. The only table read outside a tenant is the `organizations` registry itself (Phase 00.2). Pre-auth lookups avoid crossing tenants by design: Personalia's login answer names the organization, and the refresh token carries its organization as a prefix (Phase 01.1). This replaces persist DL-58's `SECURITY DEFINER` login lookup, which a local password store needed.

### Offline-capable creates (marked *offline-capable*)
- The client sends the `Idempotency-Key` header (UUID, stable for the life of the queued item), the same value as `local_id` in the body when the contract has that field, and `occurred_at`.
- A replay returns the stored response and triggers no side effects (no notifications, no outbox rows).
- `occurred_at` must be ≤ 5 min in the future and ≤ 7 days in the past.
- The helpers come from Phase 00.4.

### Security
- **No credentials.** SiAPP stores no passwords. Refresh tokens are stored only as SHA-256 hashes.
- **Logs** must never contain `Authorization` headers, tokens, NIK, passwords, camera URLs, or raw location and panic payloads (DL-17).
- **NIK** is masked in every response except where a phase says otherwise, and unmasking is audited.
- **Rate limits.** Login, refresh and panic are rate limited.
- **Integration key.** `PERSONEL_INTEGRATION_KEY` is a secret like `JWT_SECRET`: never logged, never sent to clients.

### Testing
- **Unit tests** are table-driven and cover pure logic: time, geo, tokens, resolvers, scope rules.
- **HTTP tests** go through `testkit`: a real Postgres + Redis from docker compose, one database per test package created in `TestMain`, tables truncated between tests, and the **Personel fake** (`personeltest`) in-process (`httptest`).
  - The test database is migrated as the owner. Tests connect as `siapp_app`, the same role as production, so RLS is really enforced.
  - Every endpoint gets a happy-path test, a validation-failure test, an authorization-failure test and a **cross-tenant test** (the same request with a token from another organization returns 404 and changes nothing).
  - The tests use factories and never mock the database.
- **`make check`** runs, in order:
  - `make contract` (vendor the contract and run `oapi-codegen`) and `sqlc generate`, followed by a no-diff check;
  - `go vet`;
  - `golangci-lint run`;
  - `go test -race ./...`.

## Definition of done (every phase)

1. `make check` passes locally and in CI.
2. The phase's migration and sqlc queries are committed, and the generated code is up to date.
3. New env vars appear in `.env.example` and in the config struct.
4. Every endpoint the phase builds exists in the contract, and HTTP tests validate against it. If the phase had to change the contract, the PR to `persist-ecosystem` is linked in the commit body.
5. Every new tenant table has RLS `ENABLE` + `FORCE` + the policy, and a cross-tenant test.
6. Every **⚠** item has its explanatory comment.
7. Nothing from later phases is implemented, and there are no stray `TODO`s.

## Module index (priority order)

| # | File | Phases | Personel dependency |
|---|---|---|---|
| 00 | [Platform](00-platform.md) | 5 | Seam client + stub (00.5) |
| 01 | [Identity & login](01-auth.md) | 3 | Heavy (S1, S4) |
| P | [Personalia integration](personalia-integration.md) (in `kpi-personalia-api`) | 2 | It *is* the Personalia side of S1, S4 |
| 02 | [Organization overlay](02-organization.md) | 4 | Heavy (S2, S4, S6) |
| 03 | [Access control](03-access-control.md) | 2 | Light |
| 04 | [Media](04-media.md) | 2 | None |
| 05 | [Notifications & realtime](05-notifications.md) | 3 | Events (S6) |
| 06 | [Master data](06-master-data.md) | 2 | — |
| 07 | [Attendance capture](07-attendance.md) | 7 | Heavy (S3) |
| 08 | [Location & presence](08-location-presence.md) | 2 | None |
| 09 | [Panic](09-panic.md) | 2 | None |
| 10 | [Patrol & guard posts](10-patrol.md) | 3 | None |
| 11 | [Reports](11-reports.md) | 2 | None |
| 12 | [Instructions](12-instructions.md) | 2 | None |
| 13 | [CCTV](13-cctv.md) | 2 | None |
| 14 | [Report summaries & dashboard](14-analytics.md) | 2 | Attendance read-back (S3) |
| 15 | [Assets](15-assets.md) | 2 | None |
| 16 | [Cleaning vertical](16-cleaning.md) | 3 | None |

**Build order:** file order, **except** the Personel sync group above (07.3 and the real Personel calls of 07.5), which runs after module 16.

**Out of scope here:**
- The mobile app and the web dashboard (other repos, built from the same contract).
- Everything Personel owns (see the ownership table): login screens and passwords, tenants and SaaS billing, the org hierarchy, employee records, the attendance record.
- Performance reviews (Penilaian) and payroll: deferred add-ons.
- The Platform Operator Console and the Developer Portal / Open API: not decided under the LOCK (D5, D6).
- The gap features still at brainstorm stage (client portal, stock opname, asset checklist, overtime, custom report builder, data retention, indoor presence, walkie-talkie). They need a DL and a spec first (LOCK/03 §4).
- E-Koperasi Janusa and IKI: links only (master data).

---

## Realigning the code already built

Phases 00.1–00.4 and 01.1–01.3 were built for the first plan (MySQL, local users). The last complete state is the git tag `mysql-plan-final`. **Since Phase 00.1, the code a later phase adapts is parked in `siapp/_legacy/`** (same paths as below, not compiled; see its README). Take it from there when the phase comes, and delete `_legacy/` after 01.3. Under this plan:

| Existing code | Fate |
|---|---|
| `internal/platform/config`, `clock`, `timeutil`, `httpx` (router, middleware, request id, rate limiters, decode) | **Keep**, move to `pkg/platform/`. Swap the response helpers to the contract envelopes (00.3). |
| `internal/platform/database` (MySQL) | **Rewrite** for pgx + `WithTenant` (00.2). |
| `internal/platform/testkit` | **Rewrite** for Postgres, the `siapp_app` role and the Personel fake (00.2, 00.5). |
| `internal/platform/audit` | **Keep the idea**, add `organization_id` and RLS (00.4). |
| `internal/platform/idem` (client-UUID replay) | **Replace** with the `Idempotency-Key` middleware (00.4). |
| `internal/platform/auth`: `tokens.go`, `middleware.go` (default-deny, `PublicRoutes`), `sessions.go` (rotation, reuse detection), `revocations.go`, `cleanup.go`, the device handling from 01.4 | **Keep and adapt** (01.1–01.3): sessions belong to `personnels`, carry `organization_id`, and start from Personalia's credential check instead of a local password. |
| `internal/platform/auth`: `passwords.go`, `policy.go`, `reset.go`, `users.go`, `admin_*`, `help*`, the forgot/reset/change-password endpoints, the `users` and `password_reset_tokens` tables | **Drop.** Personel owns them. |
| `internal/mail` | **Drop.** |
| `db/migrations/00001`–`00006` (MySQL) | **Replace** with new Postgres migrations from 00.2 onward. Nothing has shipped, so no data migration is needed. |
| `api/openapi.yaml` (hand-written) | **Replace** with the vendored contract (00.3). |
| Uncommitted 01.4 work | Commit it on a branch for reference, then rebuild the parts that survive in 01.3. |
