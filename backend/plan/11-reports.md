# 11: Reports (Laporan)

**Purpose:** Incident reports from field personnel, such as found items or incidents. They support visibility rules, offline submission, and search and filters.
**Manual refs:** §2.7, §3.2.3, Gambar 7, Tabel 8 (Laporan rows).
**Needs:** 03, 04, 05, 06.
**Contract operations:** `POST /reports`, `GET /reports/me`, `GET /reports/team`, `GET /reports/{id}`, `GET /admin/reports`. **Contract change:** `GET /reports/project` (the DANRU's per-project tab, Tabel 8). Export (`GET /admin/reports/export`) is built in module 14.
**Error reasons:** `REPORT_NOT_FOUND`, `TEAM_REQUIRED`.

**Who can see a report**

| Visibility | Visible to |
|---|---|
| Normal | The author, all team members, anyone whose `report.view` scope covers the team, and admins with the project in scope. |
| `only_to_danru` ("Kirim hanya ke DANRU") | The author, the people recorded in `report_audience` when the report was sent, anyone whose `report.view` scope is PROJECT for that project, and `ORG_ADMIN`s. Project admins only when they are in the audience. (Server-side, GAP-014; the web spec hides these from client-facing admins by default.) |

---

## Phase 11.1: Create and detail (offline-capable)

**Goal:** A guard files a report with photos, and the right people are notified.

**Build**
- **Table `reports`** (tenant table, RLS)
  - `id`, `organization_id`, `local_id` (uuid, UNIQUE within the organization), `personnel_id` (author), `team_id`, `project_id`, `category_id`
  - `subject` (VARCHAR(150)), `body` (text, max 5000 characters, checked by the validator)
  - `occurred_at`, `occurred_date` (DATE, project timezone), `received_at`
  - `only_to_danru` (boolean)
  - `latitude`, `longitude` (nullable; the mobile spec records GPS)
  - `submitted_offline`
  - timestamps, `deleted_at`
  - Indexes: (`organization_id`, `team_id`, `occurred_at`), (`organization_id`, `personnel_id`, `occurred_at`), (`organization_id`, `project_id`, `occurred_date`).
- **Table `report_audience`** (tenant table, RLS)
  - Columns: `organization_id`, `report_id`, `personnel_id`, with PK (`organization_id`, `report_id`, `personnel_id`) and an index on (`organization_id`, `personnel_id`).
  - Used only for `only_to_danru` reports.
  - ⚠ Comment why the audience is a snapshot: it is fixed when the report is sent, so later team changes do not expose old reports or hide them.
- **`POST /reports`** with body `{ local_id, occurred_at, team_id?, category_id, subject, body, only_to_danru, media_ids, latitude?, longitude?, submitted_offline }` (idempotent)
  - Requires `report.create`, and the team must be in scope. `team_id` may be omitted only with exactly one active membership.
  - `only_to_danru` also requires `report.create.danru_only`. The audience is `access.DanruOf(...)` (03.2), inserted in the same transaction.
  - Check the category with `masterdata.AssertCategory(…, REPORT)`.
  - Attach up to 5 media items (purpose `REPORT`). A photo taken from the gallery is accepted but flagged by `capture_source`; the manual requires camera-only, which only the app can enforce.
- **After commit (skipped on replay)**
  - Normal → `REPORT_SUBMITTED` to team members, except the author (manual Tabel 7).
  - `only_to_danru` → `REPORT_SUBMITTED` to the audience.
  - `submitted_offline` → `OFFLINE_SUBMISSION_SENT` to the author.
- **`GET /reports/{id}`**
  - Returns: author (name, photo), `occurred_at`, project name, category, subject, body, location, and media views (§3.2.3.6).
  - Apply the visibility rule. A hidden report returns `NOT_FOUND`, not 403, so its existence is not leaked.

**Tests**
- HTTP:
  - A normal report notifies the team; a DANRU-only report notifies only the DANRU.
  - A team peer cannot open a DANRU-only report; a DANRU with PROJECT scope can.
  - An offline replay is idempotent.
  - Cross-tenant: 404.

---

## Phase 11.2: Lists, search and filters

**Goal:** The "Laporan Saya", "Laporan Regu" and per-project lists, with search and filters (§2.7.1–2.7.6, §3.2.3.2–3.2.3.4).

**Build**
- **Migration**
  - `pg_trgm` GIN indexes on `reports.subject`, `reports.body` and `personnels.full_name`.
  - ⚠ Comment the search semantics: `q` matches subject, body **or author name** (the manual searches by sender name; the mobile spec dropped it, the manual wins) with `ILIKE '%' || q || '%'`, `%` and `_` escaped. Trigram indexes make this fast for 3+ characters; shorter queries fall back to a prefix match on `subject`.
- **Endpoints** (all with `httpx.ListQuery`, ordered `occurred_at DESC, id DESC`, page/limit):
  - `GET /reports/me`: reports by the caller.
  - `GET /reports/team?team_id=`: reports from teams in the `report.view` scope, optionally one team.
  - `GET /reports/project?project_id=` (contract change): requires PROJECT scope over the project.
  - `GET /admin/reports?project_id=&team_id=&category_id=&date_from=&date_to=`: admins, filtered by their project scope.
  - Filters: `q`; `date` or `month` on `occurred_date`; `category_id` (one or several).
  - ⚠ Write **one** sqlc list query that always contains the visibility predicate, with optional `sqlc.narg` filters and the team scope as an array:

    ```sql
    (NOT only_to_danru OR personnel_id = @caller OR EXISTS (SELECT 1 FROM report_audience …) OR project_id = ANY(@project_scope))
    ```

    Comment that every endpoint reuses this query and that the predicate must never be dropped.
- **List item:** `{ id, author: { name, photo_url }, occurred_at, category, subject, body_preview (first 160 chars), media_count, project_name }`.

**Tests**
- HTTP:
  - Each endpoint returns the right set.
  - Search by author name; a search shorter than 3 characters.
  - Month filter; category filter.
  - Pagination totals are correct.
  - A DANRU-only report never appears in a peer's team list.
