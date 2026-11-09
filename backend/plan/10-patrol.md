# 10: Patrol and guard posts (Patroli / Jaga)

**Purpose:** QR-tagged zones, scans that only count inside the zone's GPS radius (with checklist and photos), zone check-in/out, and DANRU progress views with approve or decline.
**Manual refs:** §2.10, §3.2.6, Gambar 10, Tabel 8 (Patroli rows).
**Needs:** 04, 05, 08 (plus `organization.Shifts` from 02.3).
**Contract operations:**
- Admin: `/admin/zones` (CRUD), `GET /admin/zones/{id}/qr`.
- Field: `GET /zones`, `GET /zones/{id}`, `POST /zones/{id}/scan`, `GET /zones/{id}/scans`, `POST /zones/{id}/check-in`, `POST /zones/{id}/check-out`, `GET /zones/{id}/live`, `GET /zones/{id}/tracking`, `GET /scans/{id}`, `POST /scans/{id}/approve`, `POST /scans/{id}/decline`, `GET /patrol/pending-approvals`.
- **Contract changes:** `GET /zones/progress`, `GET /zones/{id}/scanners`, `POST /scans/approve` (bulk), `POST /admin/zones/{id}/rotate-qr`.
**Error reasons:** `ZONE_NOT_FOUND`, `ZONE_QR_INVALID`, `ZONE_NOT_IN_PROJECT`, `ZONE_OUTSIDE_RADIUS`, `ZONE_INTERVAL_NOT_ELAPSED`, `CHECKLIST_ITEM_INVALID`, `SCAN_NOT_FOUND`, `SCAN_ALREADY_DECIDED`, `ZONE_NOT_CHECKED_IN`.

**Terms**
- **Zone types.** `PATROL` is a patrol route zone and `GUARD` is a guard post (Jaga) (DL-11; cleaning labels them "Rute Pembersihan" / "Zona Kerja Area"). Both share one flow.
- **The three circles** on the manual's zone screen map to these fields:
  - 🔴 red: required number of scans (`min_visit`); shown as a count, and as a status while below it
  - 🔵 blue: scanned but not yet decided (`submitted`)
  - 🟢 green: approved (`approved`)

---

## Phase 10.1: Zones and checklists

**Goal:** Admins define zones: location, radius, checklist and a printable QR code. Field users can list them.

**Build**
- **Table `patrol_zones`** (tenant table, RLS)
  - `id`, `organization_id`, `project_id`, `team_id` (nullable: a zone for one team or for the whole project)
  - `type` ENUM(`PATROL`,`GUARD`)
  - `name`, `alias` (nullable), `description` (nullable), `popup_text` (nullable; shown when the zone is opened, web spec)
  - `latitude`, `longitude`, `radius_meters` (default 50) — nullable together: a zone without a geofence relies on QR only (web spec 08)
  - `min_visit` (default 1), `start_date` (nullable)
  - `qr_secret` (22 chars: 16 random bytes, base64url)
  - `active`, timestamps, `deleted_at`
- **Table `patrol_checklist_items`** (tenant table, RLS): `id`, `organization_id`, `zone_id`, `label`, `sort_order`, `active`, timestamps.
- **QR payload:** `PERSIST-ZONE:1:{organizationId}:{zoneId}:{qrSecret}`.
  - ⚠ Comment why it is static, unlike the attendance QR: it is printed on stickers. Anti-fraud comes from the server-side geofence in Phase 10.2, the organization id in the payload, and rotating the secret if a sticker leaks.
- **Admin endpoints**
  - `/admin/zones?project_id=&type=`: CRUD. The checklist is part of the zone body (`checklist: [{ id?, label }]`); items removed are deactivated, never deleted.
  - `POST /admin/zones/{id}/rotate-qr` (contract change).
  - `GET /admin/zones/{id}/qr?format=png|svg`: rendered with `github.com/skip2/go-qrcode`. Set `Content-Disposition` with a filename from the zone name. (This also fills the gap the web review found: security projects need printable patrol QR codes, not only cleaning.)
- **`GET /zones?project_id=&team_id=&type=`** (project members) and `GET /zones/{id}`: zones with their active checklist. **Never** return `qr_secret`.

**Tests**
- HTTP
  - CRUD works; rotating the QR changes the payload.
  - The field-user response has no secret.
  - A non-member gets 403; another organization's zone is 404.
  - The PNG endpoint returns `image/png`.

---

## Phase 10.2: Scans and zone check-in/out (offline-capable)

**Goal:** A guard scans a zone QR and submits the checklist, notes and photos; scans count only inside the radius. Guards can also check in and out of a zone.

**Build**
- **Table `patrol_scans`** (tenant table, RLS)
  - `id`, `organization_id`, `local_id` (uuid, UNIQUE within the organization), `zone_id`, `zone_type`, `personnel_id`, `team_id`, `project_id`
  - `occurred_at`, `received_at`, `work_date` (DATE)
  - `latitude`, `longitude`, `accuracy_meters`, `distance_meters`
  - `subject`, `notes` (nullable)
  - `checklist` (jsonb: `[{ item_id, label, checked }]`)
  - `status` ENUM(`SUBMITTED`,`APPROVED`,`DECLINED`), `decided_by_id`, `decided_at`, `decision_note`
  - `submitted_offline`, timestamps
  - Indexes: (`organization_id`, `zone_id`, `team_id`, `work_date`) and (`organization_id`, `personnel_id`, `work_date`).
- **`POST /zones/{id}/scan`** (idempotent)
  - **Body:** `{ local_id, occurred_at, qr_payload, latitude, longitude, accuracy_meters, subject, notes?, checklist: [{ item_id, checked }], media_ids, submitted_offline }`
  - **Validation, in order:**
    1. Parse and verify the QR: organization, zone id (must equal the path) and secret (`hmac.Equal`). Failure: `ZONE_QR_INVALID`.
    2. Check the permission for the zone type: `patrol.scan` or `guard.scan`.
    3. The person must be a member of the zone's project (and team, when the zone has one). Failure: `ZONE_NOT_IN_PROJECT`.
    4. ⚠ **Geofence** (when the zone has one). Reject with `ZONE_OUTSIDE_RADIUS` when `distance > radius_meters + min(accuracy_meters, 30)`.
       - Comment why the tolerance is capped: a very inaccurate GPS fix must not unlock a distant zone.
       - This enforces the manual's "Anda berada diluar zona patroli" on the server, not just in the app.
       - Comment that the gap design `05-gps-indoor-floor.md` proposes warn-instead-of-block (DL-70, not adopted); keep blocking until it is.
    5. ⚠ **Patrol interval:** when the project has `patrol_interval_minutes`, a second scan of the same zone by the same person within the interval → `ZONE_INTERVAL_NOT_ELAPSED` (DL-48).
    6. ⚠ **Team and work date.** Take them from `organization.Shifts.Current(person, occurred_at)` for the zone's project, so night-shift scans count toward that shift's start date. If no shift matches, use the first active membership in the project and the business date of `occurred_at`. Comment this.
    7. **Checklist.** Every item must be active and belong to this zone. Failure: `CHECKLIST_ITEM_INVALID`. Store the labels as a snapshot. A photo is required (mobile spec SQD §4.3).
    8. **Media.** Attach with purpose `PATROL_SCAN`, in the same transaction.
  - A replay returns the stored scan and sends no notifications.
- **After commit**
  - Notify the team's DANRU: `PATROL_SUBMITTED` or `GUARD_SUBMITTED`.
  - If `submitted_offline`, send `OFFLINE_SUBMISSION_SENT` to the author.
- **Zone check-in/out** (live app; feeds the "Check-in Zona" report, DL-44)
  - **Table `zone_checkins`** (tenant table, RLS): `id`, `organization_id`, `zone_id`, `personnel_id`, `team_id`, `shift_template_id` (nullable), `check_in_at`, `check_out_at` (nullable), coordinates, timestamps. Partial UNIQUE (`organization_id`, `personnel_id`) `WHERE check_out_at IS NULL`: one open check-in per person.
  - `POST /zones/{id}/check-in` and `/check-out` (idempotent, same QR and geofence checks). Check-out without an open check-in → `ZONE_NOT_CHECKED_IN`.
  - `GET /zones/{id}/live`: who is checked in now. `GET /zones/{id}/tracking`: check-ins for a date range.

**Tests**
- Unit: radius plus tolerance at the boundary (table-driven).
- HTTP
  - An in-radius scan is accepted and the DANRU is notified; an out-of-radius scan is rejected.
  - A bad QR, a QR from another zone and a QR from another organization are rejected.
  - A checklist item from another zone is rejected.
  - A second scan inside the patrol interval is rejected.
  - An offline replay is idempotent, and the author is notified only once.
  - A night-shift scan after midnight gets the previous day's `work_date`.
  - Check-in twice without check-out fails; live shows the open check-in.

---

## Phase 10.3: Progress, approval and decline

**Goal:** The red, blue and green counters; drill-down per person; DANRU approve or decline (§2.10.5–7, §3.2.6.2, §3.2.6.3).

**Build**
- **`GET /zones/progress?project_id=&team_id=&type=&work_date=`** (contract change)
  - `work_date` defaults to today in the project timezone.
  - Returns `[{ zone, required, submitted, approved, declined }]`.
  - **Access:** members see only their own team; a DANRU can pick any team in their approve scope.
  - ⚠ Compute the counts in one query: `patrol_zones LEFT JOIN patrol_scans … GROUP BY zone`, using `count(*) FILTER (WHERE status = 'SUBMITTED')`. Comment it.
- **`GET /zones/{id}/scanners?team_id=&work_date=`** (contract change): `[{ personnel, submitted_count, approved_count }]`. This is the number shown next to each name ("angka disampingnya", §3.2.6.2).
- **`GET /patrol/pending-approvals?team_id=`**: the DANRU's queue of `SUBMITTED` scans.
- **Scan lists and detail:** `GET /zones/{id}/scans?personnel_id=&work_date=&status=` and `GET /scans/{id}`, both with the checklist and media views. Members see their own scans; a DANRU sees scans from teams in scope. Approved scans stay viewable (§3.2.6.3).
- **Decisions**
  - `POST /scans/{id}/approve`, `POST /scans/{id}/decline` `{ note }` (decline comes from the APK's `danru_patrol_feedback`; mobile spec SQD §4.6), and `POST /scans/approve` `{ ids: [] }` (max 50, contract change).
  - **Permission:** `patrol.approve` or `guard.approve`, matching the zone type, with the team in scope.
  - **Update:** `UPDATE … SET status = $new … WHERE id = ANY($ids) AND status = 'SUBMITTED'`. Rows that don't change were already decided: a single call returns `SCAN_ALREADY_DECIDED`; in bulk they are skipped and listed in the response.
  - **Audit** each decision.
  - **After commit:** notify each author **once per request**, not once per scan: `PATROL_APPROVED` / `GUARD_APPROVED` (a decline uses the same type with `data.decision = DECLINED`).

**Tests**
- HTTP
  - The counters change after a submit, an approve and a decline.
  - A member cannot decide; a DANRU of another team cannot decide.
  - A bulk approve sends one notification per author.
  - The counts in the scanners list are correct.
