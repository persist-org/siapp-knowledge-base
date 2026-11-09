# 15: Assets (Aset Manajemen)

**Purpose:** Per-project inventory (radios, uniforms, vehicles, CCTV units…) with condition, and condition/stock reports filed from the field. Not in the 2020 manual; found in the live app and adopted (DL-43, `persist-ecosystem/docs/06-traceability/05-siapp-live-discovery.md` §3.2–3.4). LOCK/03 §1 lists "Data aset" as Security-owned.
**Needs:** 03, 04, 05.
**Contract operations:** `/admin/asset-conditions` (CRUD), `GET /asset-conditions`, `/admin/assets` (CRUD), `GET /assets`, `GET|POST /asset-reports`.
**Error reasons:** `ASSET_NOT_FOUND`, `ASSET_CODE_TAKEN`, `ASSET_CONDITION_NOT_FOUND`, `ASSET_NOT_IN_PROJECT`.

**Not here:** the asset condition checklist with cases, the location leader closing cases, and stock opname. They are gap features at brainstorm stage (`gap-designs/06-inventory-stok-opname.md`, `08-data-asset.md`, DL-67/73/74 proposed). When they are adopted they extend this module on the shared "verified periodic field recording" primitive (LOCK/03 §3).

---

## Phase 15.1: Asset conditions and assets

**Goal:** Org admins keep the condition master and the per-project inventory.

**Build**
- **Table `asset_conditions`** (tenant table, RLS): `id`, `organization_id`, `name`, `sort_order`, `active`, timestamps. Partial UNIQUE (`organization_id`, `name`) `WHERE active`. Seeded per organization on first sight (insert-if-missing): Baik, Rusak Ringan, Rusak Berat, Kadaluarsa, Dalam Perbaikan.
- **Table `assets`** (tenant table, RLS), per `persist-ecosystem/docs/04-backend-core/01-data-model.md` §7A.1: `id`, `organization_id`, `project_id`, `code`, `name`, `stock` (CHECK ≥ 0), `condition_id`, `description`, `active`, timestamps. UNIQUE (`organization_id`, `code`) → `ASSET_CODE_TAKEN`.
- **Admin endpoints**: the contract's CRUD for both. A referenced condition is deactivated, not deleted (`MASTER_IN_USE`). Admins see only projects in their scope.
- **Field reads**: `GET /asset-conditions` (delta sync like 06.1), `GET /assets?project_id=` (project members, the mobile "pick project → pick asset" flow).
- **Audit** asset changes, including stock before and after.

**Tests**
- HTTP: CRUD; duplicate code; a `PROJECT_ADMIN` cannot touch another project's asset; field list for members only; cross-tenant 404.

---

## Phase 15.2: Asset reports (offline-capable)

**Goal:** A guard reports an asset's condition and stock with photos, from the field, online or offline.

**Build**
- **Table `asset_reports`** (tenant table, RLS), per data model §7A.2: `id`, `organization_id`, `local_id` (UNIQUE within the organization), `transaction_no` (UNIQUE within the organization; `AR-{yyyymmdd}-{seq}` from a per-organization sequence row locked `FOR UPDATE`), `asset_id`, `personnel_id`, `project_id`, `team_id`, `condition_id`, `stock`, `title`, `content`, `reported_at`, `submitted_offline`, timestamps.
- **`POST /asset-reports`** `{ local_id, occurred_at, asset_id, condition_id, stock?, title, content?, media_ids, submitted_offline }` (`asset.report`; idempotent)
  - The asset must belong to a project the person is a member of → otherwise `ASSET_NOT_IN_PROJECT`.
  - Attach media with purpose `ASSET_REPORT`.
  - ⚠ The report does **not** change `assets.stock` or `assets.condition_id`. Comment why: a field report is a claim until an admin acts on it (the gap designs make this a verify step; until then the admin updates the asset).
  - After commit: notify the project admins with a new `ASSET_REPORTED` type (add it to the enum and templates of 05.1); `OFFLINE_SUBMISSION_SENT` to the author when offline.
- **`GET /asset-reports?asset_id=&project_id=&date_from=&date_to=`**: field users see their teams' reports; admins see their projects. Detail includes media views.
- **Extend 14.1:** the `asset` summary type.

**Tests**
- HTTP: create with photos; replay is idempotent; an asset from another project is rejected; transaction numbers are unique under parallel creates; the asset's stock is unchanged; the `asset` summary counts the report.
