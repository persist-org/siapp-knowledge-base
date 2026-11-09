# 14: Report summaries, dashboard and audit

**Purpose:** Read-only summaries and exports for the web dashboard:
- the 13 "Ringkasan Laporan" types (DL-10 superseded by DL-44, path per DL-54)
- the overview dashboard
- Excel/PDF exports with the organization's letterhead
- the audit log browser

They become useful once there is real operational data.

**Needs:** 07, 09, 10, 11, 12 (and 00.4 audit, 06.2 letterheads). Phase 15 adds the asset type when it exists.

**Contract operations:** `GET /report-summary/{type}`, `GET /report-summary/{type}/export`, `GET /overview`, `GET /overview/charts`, `GET /admin/attendances/export`, `GET /admin/reports/export`. **Contract change:** `GET /admin/audit-logs`.

**Error reasons:** `RANGE_TOO_LARGE`, `REPORT_TYPE_UNKNOWN`.

**Rules for every endpoint in this module:**
- **Access:** admins only (`access.RequireAdmin`), filtered by their project scope. A `PROJECT_ADMIN` never sees another project's rows, even in totals.
- **Date range:** `date_from` and `date_to` are business dates and both are inclusive, computed in each project's timezone. The maximum span is 92 days; anything longer returns `RANGE_TOO_LARGE`.
- **Where the work happens:** aggregation runs in SQL, as sqlc queries with `GROUP BY`. ⚠ Every aggregate query gets a comment saying what it counts and why.
- ⚠ **Attendance numbers.** SiAPP's attendance figures come from its own captures (`attendances`). Personel holds the official record, which HR can correct there. Every attendance-based type therefore returns `source: "siapp_capture"`, and, when `personel.Attendance.Status` is reachable, a `personel_mismatch_count` per person (S3 read-back). Comment why: payroll and HR must use Personel's numbers (LOCK/01 §2); these reports are operational.

---

## Phase 14.1: The 13 summary types

**Goal:** `GET /report-summary/{type}` returns rows and totals for each type, with the filters the web spec defines (`10-report-summary-exports.md`).

**Build**
- **`pkg/reportsummary`**: a registry `map[Type]Summary`, where each `Summary` declares its filters, its columns (Indonesian headers), its sqlc query and its row mapping. `REPORT_TYPE_UNKNOWN` for anything else. The `exhaustive` linter covers the enum.
- **Types** (contract `type` enum):

  | Type | What it counts | Source tables |
  |---|---|---|
  | `patroli` | Patrol scans per zone, team and day: required vs submitted vs approved, compliance % = `least(approved / required, 1)` | `patrol_zones`, `patrol_scans` |
  | `jaga` | The same for guard posts | same, `type = GUARD` |
  | `pengganti` | Shift swaps in the range: replaced, replacing, team, shift, check-in times | `shift_swaps`, `attendances` |
  | `absensi_bulanan` | Per person and month: scheduled, present, late, leave, absent, missing, swaps covered, outside area | `team_shifts`, `team_members`, `attendances` |
  | `absensi_klien` | The same grouped by client | 🔌 needs the client grouping from Personel (S2); returns `UNPROCESSABLE` until the seam provides it |
  | `absensi_proyek` | The same grouped by project | as `absensi_bulanan` |
  | `attendance-group` | The same grouped by team (live app) | as `absensi_bulanan` |
  | `instruksi` | Instructions sent: recipients, read, feedback, per sender and category | `instructions`, `instruction_recipients` |
  | `personil` | Active people per project and team, with role and last seen | `team_members`, `personnels` (no HR fields: those are Personel's) |
  | `team_respon` | Panics: type, sender, team, occurred, resolved, minutes to resolve, report filed | `panics`, `panic_reports` |
  | `effectiveness` | Per person: attendance, shifts, leave, absences, reports, patrols, panics (live app `/form_effectiveness`) | the tables above |
  | `zone-check-in` | Zone check-ins and check-outs with shift (live app) | `zone_checkins` |
  | `asset` | Asset reports (added by Phase 15.2) | `asset_reports` |

  - ⚠ Metric definitions to comment in the queries: `late` = `check_in_at > shift_starts_at + tolerance_minutes`; `missing` = scheduled shift instances that have ended with no record (same definition as 07.4); `outside_area` = `check_in_within_area = false`.
- **Response:** `{ columns, rows, totals, source, generated_at }` inside the `SuccessEnvelope`, with page/limit over rows.

**Tests**
- **HTTP**, using a fixture month that contains every status:
  - each type's totals match the fixture;
  - a range over 92 days is rejected;
  - a `PROJECT_ADMIN` sees only their projects;
  - cross-tenant: another organization's data never appears in totals;
  - `personel_mismatch_count` reflects a difference planted in the stub.

---

## Phase 14.2: Exports, dashboard and audit browser

**Goal:** Excel and PDF exports with letterheads and photos, the overview dashboard, and a searchable audit trail.

**Build**
- **Dependencies added in this phase** (state them in the commit body): `github.com/xuri/excelize/v2` (xlsx, streaming writer) and `github.com/go-pdf/fpdf` (PDF).
- **Exports**: `GET /report-summary/{type}/export?format=xlsx|pdf|csv&include_photos=`, `GET /admin/attendances/export`, `GET /admin/reports/export`.
  - Streaming: xlsx through excelize's `StreamWriter`, flushing every 500 rows; never build the whole file in memory.
  - Headers in Indonesian; the organization's default letterhead at the top (06.2); CSV starts with a UTF-8 BOM so Excel opens it correctly.
  - ⚠ `include_photos=true` embeds thumbnails into the xlsx cells (fetched from storage, max 200 photos per file; above that, export without photos and say so in a note row). Comment why: the old system's exports came out with blank photo cells (gap design `03-custom-report-builder.md`).
  - Large exports (more than 10,000 rows) run as an asynq job that stores the file in object storage and notifies the requester with a download link (presigned, 1 h). Smaller ones stream directly.
- **Overview** — `GET /overview?project_id=` and `GET /overview/charts?year=` (web WEB-04): active people, projects and teams counts; understaffed teams (DL-02); attendance per month; patrol scans (PATROL zones only); reports per month; panics per month. All within the admin's scope.
- **Audit browser** — `GET /admin/audit-logs?entity_type=&entity_id=&actor_personnel_id=&action=&date_from=&date_to=` (`ORG_ADMIN` only; contract change).
  - Filters are optional (`sqlc.narg`); newest first; page/limit.
  - Actor names come from a `LEFT JOIN personnels` in the same query, so there is no per-row lookup.

**Tests**
- **HTTP:**
  - The xlsx opens (read it back with excelize) and has the expected rows and headers; the CSV has the BOM.
  - Photos are embedded when asked; the 200-photo cap adds the note row.
  - A large export goes through the job and the requester gets a notification with a link.
  - Overview numbers match a fixture and respect project scope.
  - The audit filter by entity returns only that entity's entries, newest first.
