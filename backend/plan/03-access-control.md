# 03: Access control (field)

**Purpose:** Turn the Satpam/DANRU permission matrix into configurable data per organization, and resolve each grant's scope (self, team or project) for every field feature. Web admins are gated separately by `access.RequireAdmin` (Phase 01.3).
**Manual refs:** §3.1, Tabel 8 (List Hak Akses User).
**Needs:** 02.
**Error reasons:** `PERMISSION_DENIED` (code `FORBIDDEN`), `PERMISSION_UNKNOWN` (code `VALIDATION_ERROR`).

**What changed from the first plan:** grants are keyed by the **team role** (`DANRU` / `MEMBER`, DL-04) instead of a position, and the matrix is per organization. "Satu tingkat di atas" (Tabel 8) becomes "the DANRU of the sender's team", as the PERSIST mobile spec simplifies it (`03-mobile-app/03-reports-instructions.md` §1).

**Model.** A **grant** is a triple: (team role, permission, scope), stored per organization.

| Scope | Covers |
|---|---|
| `SELF` | Only the person's own records |
| `TEAM` | Records of the team(s) where the person holds this role |
| `PROJECT` | Records of every team in the project(s) where the person holds this role |

**Default catalog.** Seeded from Tabel 8 for each organization on first sight; org admins can edit it afterwards.

| Permission key | MEMBER | DANRU | Manual row |
|---|---|---|---|
| `attendance.checkin` | SELF | SELF | Absen QR Code / Absen selfie |
| `attendance.qr.display` | none | TEAM | Tampil QR Code |
| `attendance.leave.request` | SELF | SELF | Pengajuan izin |
| `attendance.leave.approve` | none | TEAM | Persetujuan izin |
| `attendance.absence.create` | none | TEAM | Buat mangkir |
| `attendance.absence.view` | SELF | TEAM | Riwayat mangkir |
| `attendance.group.view` | none | PROJECT | Absensi group |
| `attendance.swap.create` | none | PROJECT | Tukar jaga |
| `report.create` | TEAM | TEAM | Buat laporan |
| `report.create.danru_only` | TEAM | none | Kirim laporan DANRU only |
| `report.view` | TEAM | PROJECT | Lihat laporan regu / per proyek |
| `instruction.create` | none | PROJECT | Buat instruksi / kirim ke banyak regu/proyek |
| `team.positions.view` | none | PROJECT | Lihat posisi anggota (regu / per anggota / per proyek) |
| `patrol.scan` | TEAM | TEAM | Membuat laporan patroli |
| `patrol.approve` | none | TEAM | Approve laporan patroli |
| `guard.scan` | TEAM | TEAM | Membuat laporan jaga |
| `guard.approve` | none | TEAM | Approve laporan jaga |
| `cctv.view` | PROJECT | PROJECT | Mengakses CCTV |
| `panic.create` | TEAM | TEAM | Membuat sinyal panik |
| `panic.view` | TEAM | PROJECT | Sinyal panik pribadi / grup / per proyek |
| `panic.finish` | SELF | TEAM | Menyelesaikan sinyal panik (DL-31) |
| `asset.report` | TEAM | TEAM | Laporan aset (live app, DL-43) |
| `ememo.use` | TEAM | TEAM | E-Memo (cleaning only, module 16) |

**Always allowed, so not in the catalog:**
- own attendance view;
- own instruction inbox and feedback;
- the team directory (Phase 02.4);
- receiving panic alerts.

---

## Phase 03.1: Permission catalog and matrix

**Goal:** Store grants per organization and team role, seed them with the defaults above, and let org admins edit them.

**Build**
- **Catalog.** `pkg/access/catalog.go`:
  - `type Permission string` with one constant per key;
  - `type Scope string` with `SELF`, `TEAM`, `PROJECT`;
  - a `Catalog` slice of `{ Key, Module, Description (Indonesian), Defaults map[TeamRole]Scope, BusinessLines []BusinessLine }` built from the table above. `ememo.use` is `cleaning` only.
  - `IsKnown(key)`.
- **Table `role_permissions`** (tenant table, RLS):
  - columns `organization_id`, `team_role` ENUM(`DANRU`,`MEMBER`), `permission` (VARCHAR(64)), `scope` ENUM(`SELF`,`TEAM`,`PROJECT`), timestamps;
  - primary key (`organization_id`, `team_role`, `permission`).
- **Seeding.** ⚠ `access.EnsureDefaults(ctx, orgID)` runs on first sight of an organization (called from `identity.Exchange` after the organization upsert) and from `cmd/seed`. It inserts only the missing defaults with `INSERT … ON CONFLICT DO NOTHING`. Comment why: existing rows are never overwritten, so admin edits survive a re-seed, and new catalog keys reach existing organizations.
- **Admin endpoints** (`ORG_ADMIN` only; contract change: these replace the web-only `/admin/roles` permission matrix for field roles):
  - `GET /admin/field-permissions` returns the catalog plus the matrix for both team roles.
  - `PUT /admin/field-permissions/{team_role}` with `{ grants: [{ permission, scope }] }` replaces all grants of that role in one transaction. An unknown key returns `PERMISSION_UNKNOWN`. Audit the change.
- **`GET /auth/me`** gains `permissions: [{ permission, scope }]`, merged across today's active memberships, keeping the widest scope per key, and dropping keys whose business line does not match any of the person's projects. The mobile app shows or hides menus with it.

**Tests**
- **Unit:** scope merging (PROJECT beats TEAM, TEAM beats SELF).
- **HTTP:**
  - a new organization gets the defaults;
  - a re-seed keeps an admin's change;
  - replacing grants works;
  - `/auth/me` differs correctly for a member and a DANRU;
  - `ememo.use` appears only for someone in a cleaning project;
  - cross-tenant: org A's edit does not change org B's matrix.

---

## Phase 03.2: Access service and middleware

**Goal:** One place that answers "may this person do X, and on which teams?" Every feature module from 07 onward uses it.

**Build**
- **`access.Service`** (exported, with doc comments).
  - **`Resolve(ctx, personnelID, perm, on timeutil.Date) (Resolution, error)`**
    - `Resolution` holds `Allowed`, `SelfOnly bool`, `TeamIDs`, `ProjectIDs []int64`, and a method `HasTeam(id)`.
    - ⚠ Comment the rules. For each active membership whose role grants the permission:
      - `SELF` → only set `SelfOnly`;
      - `TEAM` → add that team;
      - `PROJECT` → add every active team of that project, and the project itself.
    - `SelfOnly` is true only when **no** grant is wider than SELF.
    - Implement it with two sqlc queries: the grants joined to memberships, then the teams of the PROJECT-scope projects.
  - **`AssertTeam(ctx, personnelID, perm, teamID, on)`** returns `PERMISSION_DENIED` when the team is not in scope.
  - **`DanruOf(ctx, teamID, on) ([]int64, error)`** returns the team's DANRU on that date. ⚠ This implements "Kirim laporan DANRU only" (Tabel 8). Comment the fallback: when the sender is the DANRU, or the team has none that day, return the `PROJECT_ADMIN`s of the project and the `ORG_ADMIN`s. An empty result is allowed.
- **`access.Require(perm)` middleware.**
  - Resolves for today in `Asia/Jakarta`. This coarse check only gates the route; usecases re-check with the correct project date.
  - Rejects with `PERMISSION_DENIED` when `!Allowed`.
  - Stores the `Resolution` in the context. Usecases read it via `access.From(ctx)` and never resolve twice.
- **Business-line gate.** `access.RequireBusinessLine(line)` for cleaning-only routes (module 16): the target project must have that `business_line`, otherwise `FORBIDDEN` with reason `FEATURE_NOT_ENABLED`.
- **Admins** (web roles) never go through `Require`; they only use `/admin` routes. Comment this on `Require`.

**Tests**
- **Unit** (`Resolve`, against the test database with fixture memberships):
  - a member in 1 team;
  - a DANRU of 2 teams in 2 projects (PROJECT expansion);
  - a SELF-only grant;
  - no grant, so `Allowed = false`;
  - an ended membership is ignored.
- **Unit** (`DanruOf`): the team's DANRU; the sender is the DANRU → project admins; no DANRU that day → project admins.
- **HTTP:** a test route guarded by `attendance.qr.display` returns 403 for a member and 200 for a DANRU; a cross-tenant team id is never in scope.
