# 02: Organization

**Purpose:** SiAPP owns the operational hierarchy: koordinator wilayah → kantor cabang → klien → proyek → regu. It also owns the people in each regu with their field role, and the shifts. Every feature is scoped by it. People and their employment stay in Personalia (`users`, `m_employee_work`).

**Hierarchy** (master data, from `docs/masterdata/*.xlsx`):

| Level | Table | Code (the id) | Example | Name column |
|---|---|---|---|---|
| Koordinator wilayah | `regions` | `OS{n}` | `OS1` | `description` ("Outsourcing 1") |
| Kantor cabang | `branches` | `{n}{K\|I\|P}`, `n` = its region's number | `1K` | `description` |
| Klien | `clients` | `{branch}-{company_prefix}` | `1K-IBMI` | `description` |
| Proyek | `projects` | `{client}-{S\|C}` (`S` security, `C` cleaning) | `1K-IBMI-S` | `description` |
| Regu | `teams` | `{project}{n}`, `n` from 1, per project | `1K-IBMI-S1` | `description` |

The branch letter is the employing PT: `K` = PT Kharisma Potensia Indonesia (KPI), `I` = PT Insan Daya Sentosa (IDS), `P` = PT Potensial Cipta Daya (PCD).

**Who sees what** (D17, D18):
- **Tenant:** the whole group (KPI + IDS + PCD) is **one** organization. The PT is a level inside it, not a tenant.
- **`ORG_ADMIN`** (super admin): everything.
- **`REGION_ADMIN`**: everything under one koordinator wilayah, across all three PTs.
- **DANRU** (field): the proyek of the regu where they are DANRU. A **MEMBER** sees the proyek of their own regu.

**LOCK:**
- ⚠ **Overrides F4** for projects and locations (D17): the hierarchy is SiAPP master data, no longer read through from Personel. Personel still owns people, employment and HR profiles.
- LOCK/03 §1: Security owns the regu, post and shift overlay.
- D1: regu, members and shifts live in SiAPP, which is the source of truth. D9 (what Personel pulls back) still applies, and now covers the hierarchy too.

**Manual refs:** §2.9 Regu, §3.2.5, Gambar 9.

**Needs:** 01.

**Contract operations:**
- Admin: `GET|POST /admin/regions`, `GET|PATCH|DELETE /admin/regions/{code}`, and the same for `/admin/branches`, `/admin/clients`, `/admin/projects` and `/admin/teams`. Also `GET|POST /admin/teams/{code}/members`, `DELETE /admin/teams/{code}/members/{personnel_id}` and `POST /admin/master-data/import/{sheet}`.
- Field: `GET /teams/me`, `GET /teams`, `GET /teams/{code}/members`, `GET /personnels/{id}`.
- **Contract changes:** the regions, branches and clients endpoints, `POST|DELETE /admin/projects` (projects are now created here; D5 no longer applies), path ids become codes, the import endpoint, and shift templates and team schedules (02.5).

**Error reasons:** `REGION_NOT_FOUND`, `BRANCH_NOT_FOUND`, `CLIENT_NOT_FOUND`, `PROJECT_NOT_FOUND`, `TEAM_NOT_FOUND`, `CODE_TAKEN`, `BRANCH_REGION_MISMATCH`, `BRANCH_LETTER_INVALID`, `CLIENT_PREFIX_INVALID`, `HAS_ACTIVE_CHILDREN`, `PROJECT_NOT_CONFIGURED`, `MEMBERSHIP_NOT_FOUND`, `MEMBERSHIP_OVERLAP`, `TEAM_HAS_DANRU`, `SHIFT_NOT_FOUND`, `SHIFT_GROUP_NOT_FOUND`, `SHIFT_INVALID_TIME`, `SCHEDULE_EXISTS`, `NOT_PROJECT_MEMBER`, `AUTH_NOT_IN_GROUP`.

**What changed from the previous plan:**
- Projects are no longer a read-through of Personel projects/locations. The five levels are SiAPP master data, entered by admins or imported from the sheets.
- The tenant is the group, not the employing PT (D17 replaces D11's tenant rule).
- `PROJECT_ADMIN` is replaced by `REGION_ADMIN` (D18).
- Codes are the ids. There are no numeric ids for these five tables (D19).
- One proyek has one address, timezone and GPS point, and all its personnel work there. Regu have no geofence of their own (D20).
- The Personel placement check (`PERSON_NOT_PLACED`) is dropped: regu membership is decided in SiAPP.
- `positions` (GUARD / SQUAD_LEADER) stay gone. The field role is `team_members.role` ∈ {`DANRU`, `MEMBER`} (DL-04).

---

## Phase 02.1: Group tenant and region admins

**Goal:** Every KPI, IDS and PCD employee lands in one organization, and an admin can be scoped to one koordinator wilayah. This changes code built in phase 01, so it comes first.

**Build**
- **Tenant mapping** (`pkg/platform/personel`, the one mapping function D11 points to):
  - Config `GROUP_COMPANY_IDS=3,45,89`: the Personel `m_companies.id` of KPI (3), IDS (45) and PCD (89). Config `GROUP_ORGANIZATION_ID`: the SiAPP organization they all map to.
  - A person whose employing `company_id` is not in the list cannot log in: `AUTH_NOT_IN_GROUP`. Comment why: SiAPP serves only this group.
  - Keep the person's own PT (`company_id`) and `client_id` on `personnels` for display.
  - Migration: move the rows of the three per-PT organizations into the group organization. Nothing is live yet, so a reset of dev data is acceptable instead.
- **Table `regions`** (tenant table, RLS). It lives here because `REGION_ADMIN` points at it.
  - `organization_id`, `code` (`^OS[1-9][0-9]*$`), `description`, `active`, timestamps, `deleted_at`
  - PRIMARY KEY (`organization_id`, `code`)
- **Web roles** (changes 01.3's `web_role_assignments`):
  - `web_role` ENUM becomes (`ORG_ADMIN`, `REGION_ADMIN`). `PROJECT_ADMIN` is removed.
  - `project_ids` is replaced by `region_code` (nullable, FK to `regions`).
  - CHECK: `ORG_ADMIN` has no region, `REGION_ADMIN` has exactly one.
  - ⚠ `access.RequireAdmin` resolves an admin's scope to a set of project codes: all of them for `ORG_ADMIN`, and the projects under the region for `REGION_ADMIN`. Every admin query filters by that set, so later modules keep their "admin sees their projects" checks unchanged.
- **Admin endpoints:** `/admin/regions` CRUD (`ORG_ADMIN` only). Delete is a soft delete, refused with `HAS_ACTIVE_CHILDREN` while branches that are not deleted exist, active or inactive.
- **Audit:** `region.create|update|delete`, and the web role changes.

**Tests**
- HTTP:
  - KPI, IDS and PCD employees all get the same `organization_id` in their token. Someone from another PT gets `AUTH_NOT_IN_GROUP`.
  - A `REGION_ADMIN` of `OS1` sees `OS1`'s data from all three PTs and gets 403 on `OS2`'s.
  - A `REGION_ADMIN` cannot manage regions or web roles.
  - Granting `REGION_ADMIN` without a region, or `ORG_ADMIN` with one, is rejected.

---

## Phase 02.2: Branches, clients, projects and teams

**Goal:** Admins maintain the rest of the hierarchy, with codes generated and validated by the rules in the table above.

**Build**
- ⚠ **Codes** (D19): the code is the primary key (`organization_id`, `code`), and child tables reference their parent by (`organization_id`, `<parent>_code`). The server builds each code from the inputs. **It cannot be edited after creation**: it contains its parent's code, so a change would have to rename every child. Comment that rule on each table.
- **Table `branches`** (kantor cabang)
  - `code`, `region_code`, `company_letter` ∈ {`K`,`I`,`P`}
  - `manager_name`, `city`, `phone`, `fax`, `email`, `address`, `zip_code`, `description` (sheet columns, all nullable except `description`)
  - `active`, timestamps, `deleted_at`
  - ⚠ The code is `{region number}{company_letter}`: `1K` must sit under `OS1`. A digit that does not match the region → `BRANCH_REGION_MISMATCH`. Any other letter → `BRANCH_LETTER_INVALID`. The client sends `region_code` and `company_letter`, and the server builds the code.
- **Table `clients`** (klien). Columns: PJ48 `m_companies` without `company_type` (the table says it) and without `company_name` (the description is the name), plus the sheet's columns.
  - `code`, `branch_code`
  - `description` (the name), `company_code` (entered by hand, not generated), `company_prefix` (`^[A-Z0-9]{2,10}$`, otherwise `CLIENT_PREFIX_INVALID`), `company_bidang_usaha`
  - `director_name`, `city`, `phone`, `fax`, `address`, `zip_code`, `email`
  - `active`, `created_by`, `modify_by`, timestamps, `deleted_at`
  - The code is `{branch_code}-{company_prefix}`.
- **Table `projects`** (proyek). Same `m_companies` columns, plus the sheet's columns, plus SiAPP's operating settings.
  - `code`, `client_code`
  - `description`, `company_code`, `company_prefix` (nullable), `company_bidang_usaha`
  - `business_line` ENUM(`security`,`cleaning`) (DL-20, LOCK/03 §2). It sets the code suffix, `S` or `C`, so a client has at most one project per business line.
  - `city`, `address`, `phone`, `email`
  - `timezone` (must be in `timeutil.SupportedTimezones`; the sheet's WIB/WITA/WIT map to `Asia/Jakarta`/`Asia/Makassar`/`Asia/Jayapura`) (DL-03)
  - `latitude`, `longitude` (`numeric(9,6)`, nullable), `attendance_radius_meters` (default 150), `strict_attendance_geofence` (default false)
  - `gps_tracking_enabled` (default true), `patrol_interval_minutes` (nullable; null = not limited) (DL-48)
  - `emergency_contacts` jsonb (module 06's per-project override, DL-14)
  - `next_team_number` (default 1), `active`, `created_by`, `modify_by`, timestamps, `deleted_at`
  - ⚠ A project without coordinates is **not configured**. A feature used there returns `PROJECT_NOT_CONFIGURED`, so a guard cannot check in at a site whose location is unknown. The sheets have no coordinates, so imported projects start unconfigured.
- **Table `teams`** (regu)
  - `code`, `project_code`, `description` (the name), `phone`, `min_personnel` (default 0; understaffed when the active member count is below it, DL-02), `notes` (the sheet's unnamed last column, such as "tidak ada danru & chief")
  - `validate_shift` (default true), `reject_attendance_after_tolerance` (default false) (DL-48)
  - `relationship_officer_personnel_id` (nullable, FK `personnels`)
  - `active`, timestamps, `deleted_at`
  - ⚠ The code is `{project_code}{n}`. `n` comes from `UPDATE projects SET next_team_number = next_team_number + 1 … RETURNING`, in the same transaction as the insert. Numbers are **never reused**: after `S2` is deleted, the next team is still `S5`. Comment why: past attendance and reports keep pointing at the old code.
- **Admin endpoints:** CRUD for `/admin/branches`, `/admin/clients`, `/admin/projects` and `/admin/teams`, each with `q`, `active` and parent filters (`project` also takes `business_line`).
  - A `REGION_ADMIN` only sees and writes inside their region.
  - PATCH rejects the code and the fields it is built from (parent, letter, prefix, business line).
  - Delete is a soft delete, refused with `HAS_ACTIVE_CHILDREN` while children that are not deleted exist, active or inactive (an inactive child let through would outlive its parent; changed in the pre-PR review, 2026-10-21). Deleting a team also ends its memberships today, including ones planned to end later.
- **`organization.Projects`** (exported): `Get(ctx, code)`, `Location(ctx, code) (*time.Location, geo.Point, radius, error)` (returns `PROJECT_NOT_CONFIGURED` when unset), and `Ancestors(ctx, code)` (client, branch, region) for reports.
- **Audit:** every create, update and delete, with before and after values.

**Tests**
- Unit (table-driven): the code builders and validators: `OS1` + `K` → `1K`; `2K` under `OS1` rejected; letter `X` rejected; prefix rules; project suffix by business line.
- HTTP:
  - Create the chain `OS1` → `1K` → `1K-IBMI` → `1K-IBMI-S` → `1K-IBMI-S1`, `1K-IBMI-S2`. Delete `S2`, create again → `S3`.
  - A second security project for the same client → `CODE_TAKEN`.
  - Patching a code, parent, letter, prefix or business line is rejected. Patching the description is allowed.
  - Deleting a branch with active clients → `HAS_ACTIVE_CHILDREN`.
  - A `REGION_ADMIN` of `OS1` cannot read or create under `OS2`.
  - An unconfigured project → `PROJECT_NOT_CONFIGURED` from `Location`.

---

## Phase 02.3: Team members

**Goal:** Admins assign people to a regu with a field role. History is kept, so past records keep their original regu.

**Build**
- **Table `team_members`** (tenant table, RLS)
  - `id`, `organization_id`, `team_code`, `personnel_id`, `role` ENUM(`DANRU`,`MEMBER`)
  - `start_date` DATE, `end_date` DATE (nullable, inclusive), timestamps
  - ⚠ Exclusion constraint: no overlapping memberships of one person in one team: `EXCLUDE USING gist (organization_id WITH =, team_code WITH =, personnel_id WITH =, daterange(start_date, coalesce(end_date, 'infinity'), '[]') WITH &&)` → `MEMBERSHIP_OVERLAP`. Comment that a person may hold memberships in several teams and projects at once (§2.9.1), just not two overlapping ones in the same team.
  - ⚠ One active DANRU per team per date (BR-RULE-02): a second exclusion constraint on (`organization_id`, `team_code`, date range) `WHERE role = 'DANRU'` → `TEAM_HAS_DANRU`.
- People come from Personalia `users`. A person is added by NIK and resolved with `identity.People.Ensure`, so someone who never signed in can be added.
- **Admin endpoints**
  - `POST /admin/teams/{code}/members` `{ nik, role, start_date }`.
  - `GET /admin/teams/{code}/members?on_date=`.
  - `DELETE /admin/teams/{code}/members/{personnel_id}`: sets `end_date` to today in the project timezone (never deletes the row).
  - **Contract changes:** `POST /admin/team-members/{id}/move` `{ to_team_code, effective_date }` and `/change-role` `{ role, effective_date }`. Each ends the old row on `effective_date - 1` and inserts a new one, in one transaction. Comment why history is preserved instead of overwritten.
- **`organization.Query`** (exported, with doc comments)
  - `ActiveMemberships(ctx, personnelID, on timeutil.Date)`: joins team and project.
  - `TeamMembers(ctx, teamCode, on)`, `TeamDanru(ctx, teamCode, on)`
  - `IsMemberOfProject(ctx, personnelID, projectCode, on)`, `IsDanruOfProject(ctx, personnelID, projectCode, on)`
  - `TeamWithProject(ctx, teamCode)`
- **Audit:** `team_member.add|move|change_role|end`.

**Tests**
- Unit: none (the constraints do the work); HTTP tests cover them.
- HTTP:
  - Add, then move: history has 2 rows, and the active query returns the new team.
  - An overlap in the same team is rejected; memberships in 2 projects are allowed.
  - A second DANRU on the same dates is rejected.
  - A person who never signed in can be added by NIK.
  - A `REGION_ADMIN` cannot add members to a team outside their region.

---

## Phase 02.4: Master data import

**Goal:** Load the sheets in `docs/masterdata` (and later updates of them) without typing each row.

**Build**
- `POST /admin/master-data/import/{sheet}` with `sheet` ∈ {`regions`, `branches`, `clients`, `projects`, `teams`, `members`}, an `.xlsx` upload in the sheets' column layout, and `?dry_run=true`. `ORG_ADMIN` only.
  - Every row goes through the same validation as the CRUD endpoints. The response lists each row's result: created, updated (matched by code), or the error with its row number.
  - ⚠ All rows in one transaction: if any row fails, nothing is saved. Comment why: a half-loaded hierarchy is worse than none.
  - Import order follows the hierarchy. A row whose parent is missing fails with that parent's `*_NOT_FOUND`.
  - Team codes come from the sheet, and `projects.next_team_number` is raised past the highest imported number.
  - `members` (the Personil sheet): only `NIK`, `Regu` and `Jabatan` are read. `Jabatan` "Danru Security" → `DANRU`, anything else → `MEMBER`. `start_date` is the import date. The other columns are ignored, because Personalia owns the HR profile.
  - Timezone text that is not exactly one of WIB/WITA/WIT is a row error. The current Proyek sheet has one: `1K-SUJA-S` lists two zones, so that row must be fixed before import.
- Command `cmd/import-masterdata` runs the same import from files, for the first load.

**Tests**
- HTTP:
  - Import the six sample sheets in order (with `1K-SUJA-S` fixed); counts match the sheets.
  - A dry run saves nothing. One bad row rolls back the whole sheet and reports its row number.
  - Importing the same sheet twice updates rows and creates no duplicates.
  - The `1K-SUJA-S` row as it is now → a timezone row error.

---

## Phase 02.5: Shifts and schedules

**Goal:** Per-person shifts and schedules that work the same way as PJ48 (Persist Personalia admin panel): a shift master, weekly shift groups, each person's FIXED or FLEXIBLE assignment, and one schedule row per person per date. Then resolve a person's concrete shift instances. (D1 may later move the source to Personel's `m_harikerja`, so keep it behind the `organization.Shifts` interface.)

**Decisions (2026-10-20):** replaces the earlier per-project templates and per-team schedule (`shift_templates`, `team_shifts`). Masters are org-wide and ORG_ADMIN-only, as in PJ48. SiAPP has no contract end date, so FIXED schedules use a rolling window instead of PJ48's contract range. A shift's times are local to the project of the person's regu on that date.

**Build**
- **Table `shifts`** (PJ48 `m_shifts`): `name` (unique per org among live rows), `start_minute`, `end_minute` (0–1439; equal → `SHIFT_INVALID_TIME`), `is_off` (an OFF day has no times, overnight or tolerance, and refuses them; PJ48 matched on the name `OFF`), `tolerance_minutes` (default 15, shorter than the shift), `active` (refused while a group uses the shift), soft delete. A shift that ends before it starts is **overnight**, and the request must say `overnight: true`: otherwise it is `SHIFT_INVALID_TIME` (a reversed range is usually a typo). The API speaks `"HH:mm"`.
- **Tables `shift_groups` + `shift_group_days`** (PJ48 `m_shift_group_master` + `t_shift_group_detail`): a weekly pattern, always seven days (ISO weekday 1–7) each pointing at a shift. Editing a group changes the pattern only; generated schedules stay (as in PJ48).
- **Table `personnel_shift_assignments`** (PJ48 `m_employee_work.shift_groub_type` / `shift_groub`): one row per person, `schedule_type` FIXED (needs a group) or FLEXIBLE (none), `effective_date`, `generated_until`.
- **Table `schedules`** (PJ48 `m_schedules`): `personnel_id`, `work_date` (the date the shift starts, project timezone), `shift_id`, `shift_group_id` (source; NULL = entered or changed by hand). UNIQUE (person, date) → `SCHEDULE_EXISTS`.
- **Generation** (PJ48 `ScheduleGeneratorService`): setting FIXED deletes from the earlier of the new and old effective dates and generates `effective_date .. today + 60`. Past dates are never touched; today is locked once its shift has started (Phase 07 adds "already checked in"). FLEXIBLE generates and deletes nothing. The nightly worker task `organization:extend-schedules` (00:05 WIB) extends every FIXED window and never overwrites a manual entry.
- **Admin endpoints:** `/admin/shifts` and `/admin/shift-groups` CRUD (writes ORG_ADMIN only), `GET|PUT /admin/personnel/{id}/shift-assignment`, `POST /admin/personnel/{id}/schedules` (1–62 dates, all or none), `GET /admin/schedules?date_from&date_to` (≤ 62 days, `team_code`, `personnel_id`, `q`), `GET|PATCH|DELETE /admin/schedules/{id}`, `POST /admin/schedules/bulk-delete`. A REGION_ADMIN reaches a person who serves in a regu of their region on the schedule's date (for the assignment: a membership that has not ended).
- **Timezone:** the project of the person's regu on that date. Projects in two timezones on one day → `SCHEDULE_TIMEZONE_AMBIGUOUS`. No regu → WIB, no site. A FIXED assignment checks every date it would generate. A person Personalia reports as inactive gets no new assignment or schedule (`PERSONNEL_INACTIVE`), and the nightly extension skips them.
- **`organization.Shifts`** (exported interface + `LocalShifts`): `Instances(ctx, org, personnelID, from, to)` → `ShiftInstance` {ScheduleID, WorkDate, Shift, IsOff, ToleranceMinutes, Location, ProjectCodes, StartsAt, EndsAt (UTC)}; `Current(ctx, org, personnelID, at)` keeps working shifts whose `[StartsAt − 2h, EndsAt + 4h]` contains `at`, looking at yesterday, today and tomorrow.
- **Not yet:** PJ48's Excel schedule import (preview, then apply) belongs with 02.4. Attendance re-sync on a schedule change (PJ48 `ScheduleSyncService`) comes with Phase 07.

**Error reasons:** `SHIFT_NOT_FOUND`, `SHIFT_GROUP_NOT_FOUND`, `SHIFT_INVALID_TIME`, `SHIFT_NAME_TAKEN`, `SHIFT_GROUP_NAME_TAKEN`, `SHIFT_IN_USE`, `SHIFT_GROUP_IN_USE`, `SHIFT_ASSIGNMENT_NOT_FOUND`, `SCHEDULE_NOT_FOUND`, `SCHEDULE_EXISTS`, `SCHEDULE_TIMEZONE_AMBIGUOUS`.

**Tests**
- Generation: weekly pattern from today to today + 60; the past and a started today are kept; a new group replaces from tomorrow; FLEXIBLE changes nothing; the nightly extension adds the next day and keeps manual entries.
- Manual: several dates at once; an occupied date refuses the whole request; move/change; bulk delete all or none.
- Instances: a night shift in WIT ends the next morning; `Current` from 2 h before to 4 h after; OFF days have no times; two timezones on one day are refused.
- Scope: a REGION_ADMIN of OS1 gets 403 on people of OS2 for every action.

---

## Phase 02.6: Directory endpoints (field users)

**Goal:** The Regu screens: my teams, teams in a project, team detail and member biodata (§2.9, §3.2.5.1–3.2.5.5).

**Decisions (2026-10-21):** "today" is each project's own business date. `GET /teams` always returns a flat list of teams (with `project_code`/`project_name`, `is_my_team`, `my_role`); the app groups by project. "Today's shifts" are the working shifts scheduled today for the regu's members (02.5 made schedules per person), with a count each, and each member carries their own `today_shift`. A card lists the target's regu in the projects both share (all of them for one's own card). `NOT_PROJECT_MEMBER` is 403 `SCOPE_DENIED`, also for a project that does not exist. `position_name` is cached on `personnels` from Personalia on every login and lookup (migration 00013).

**Build**
- ⚠ **Field scope:** a person sees the projects where they hold an active membership. A DANRU sees their project because they are its regu's DANRU, and a MEMBER sees the project of their own regu. There is no separate project assignment.
- `GET /teams/me` ("Lihat Regu Saya", §3.2.5.5): for each active membership, the team detail below.
- `GET /teams?project_code=`: teams of a project; requires `IsMemberOfProject`, otherwise `NOT_PROJECT_MEMBER`. Without `project_code`, the projects of the caller's active memberships with their teams (§2.9.1–2.9.2).
- `GET /teams/{code}/members`: team detail for members of the team's project.
  - Project, client and team names, relationship officer, DANRU, today's shift(s) for the project's business date, `understaffed` (DL-02).
  - `members: [{ personnel_id, full_name, role, position_name, photo_url }]`, DANRU first, then by name. `position_name` comes from Personalia (cached), for display only.
- `GET /personnels/{id}`: `full_name`, `position_name`, `phone`, `team_name`, `project_name`, `photo_url`, NIK masked.
  - Only visible when the requester and the target share a project.
  - No birth date and no other HR data. Comment why: Personalia owns the HR profile; SiAPP shows only what the Regu screen needs.
- Do **not** add online status here. Presence comes in Phase 08.1.

**Tests**
- HTTP:
  - A member sees their team; a non-member gets 403 `NOT_PROJECT_MEMBER`.
  - A DANRU of `1K-SUJA-S2` sees project `1K-SUJA-S` and not `1K-PRST-S`.
  - Today's shift uses the project timezone.
  - "My teams" returns both teams for a person in 2 projects.

---

## Knock-on changes in other plans

These still describe the old model and need updating before their phases start:
- **01-auth** (01.1, 01.3) and `docs/pr/siapp-auth.md`: tenant = employing PT, and `PROJECT_ADMIN` with `project_ids`. Phase 02.1 changes both.
- **03, 06, 11, 14, 15, 16**: `PROJECT_ADMIN` checks become "admin scope contains the project", and `project_id` becomes `project_code`. 16 depends on 02.1 for `business_line`; the type now comes from 02.2.
- **README**: the ownership table (projects no longer read from Personel) and seam S2 (projects and placements are no longer needed from Personel; positions and levels still are, for display).
- **persist-ecosystem LOCK** F4 and the decision log: record D17–D20.
