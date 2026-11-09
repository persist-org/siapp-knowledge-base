# 07: Attendance capture (Absensi)

**Purpose:** Everything the field does about attendance. It covers:
- selfie and DANRU-QR check-in/out
- **pushing every attendance event to Persist Personel**, which holds the official record
- attendance history
- leave, permit and sick requests (izin, cuti, sakit), **owned by SiAPP** (07.5)
- absence marking (mangkir) and the group view
- shift swaps (tukar jaga)

**Priority:** the SiAPP → Personel sync is **built last** (README, "Personel sync is built last"): Phase 07.3 runs after module 16. Capture and the outbox rows (attendance and leave) are built in order. Leave itself is SiAPP's and does not wait.

**LOCK:** F5. "Absensi (system-of-record) → Personel; aplikasi = alat capture → kirim ke Personel." Leave requests were Personel's under LOCK/01 §2. SiAPP owns them now (07.5, D2, D21).

**Manual refs:** §2.6, §3.2.2, Gambar 5–6, Tabel 8 (Absensi rows).

**Needs:** 02, 03, 04, 05, 06.

**Contract operations:**
- Capture: `POST /attendances/selfie/check-in|check-out`, `POST /attendances/qr/check-in|check-out`, `PATCH /attendances/{id}/check-out`, `GET /attendances/qr/display`.
- Views: `GET /attendances/me`, `GET /attendances/{id}`, `GET /attendances/replaced`, `GET /attendances/replacing`, `GET /attendances`.
- DANRU: `POST /attendances/absent`, `POST /attendances/group`, `GET|POST /permissions`, `GET /permissions/me`, `POST /permissions/{id}/approve|decline`, `GET /permission-categories`, `GET|POST /shift-swaps`.
- Admin: `GET /admin/attendances`, `PATCH /admin/attendances/{id}`.
- **Contract changes:** `GET /attendances/today`, `GET /attendances/group-summary`, `DELETE /attendances/absent/{id}`, `DELETE /shift-swaps/{id}`, `GET /shift-swaps/replaceable`, `GET /shift-swaps/candidates`, `GET /permissions/{id}`, `GET /permissions/quota`, `GET /permissions/approver`, `POST /permissions/{id}/cancel`, `GET|POST /admin/permission-categories`, `GET|PATCH /admin/permission-categories/{code}`, `PUT /admin/permission-categories/{code}/quotas`, `PUT /admin/leave-client-settings/{client_code}`, `GET|POST /admin/personnel/{personnel_id}/leave-ledger`, `GET /admin/permissions`, `POST /admin/permissions/{id}/approve|decline`; the old `/permissions` bodies and the `Permission` schema are rewritten for 07.5; `personel_sync_status` on the attendance schema.

**Error reasons:**
- Check-in/out: `ATTENDANCE_NO_SHIFT`, `ATTENDANCE_SHIFT_NOT_ACTIVE`, `ATTENDANCE_ALREADY_CHECKED_IN`, `ATTENDANCE_NOT_CHECKED_IN`, `ATTENDANCE_NOT_ALLOWED`, `ATTENDANCE_OUTSIDE_AREA`
- QR: `ATTENDANCE_QR_INVALID`, `ATTENDANCE_QR_EXPIRED`, `ATTENDANCE_QR_WRONG_TEAM`, `ATTENDANCE_QR_UNAVAILABLE`
- Leave: `LEAVE_NOT_FOUND`, `LEAVE_ALREADY_DECIDED`, `LEAVE_CATEGORY_INVALID`, `LEAVE_OVERLAP`, `LEAVE_QUOTA_EXCEEDED`, `LEAVE_SCHEDULE_MISSING`, `LEAVE_NO_WORKING_DAYS`, `LEAVE_ALREADY_CANCELLED`, `LEAVE_CANNOT_CANCEL`, `LEAVE_NO_APPROVER`
- Absence: `ABSENCE_ALREADY_PRESENT`
- Group: `GROUP_ATTENDANCE_DISABLED`
- Swap: `SWAP_NOT_FOUND`, `SWAP_REPLACED_NOT_ABSENT`, `SWAP_ALREADY_EXISTS`, `SWAP_SHIFT_OVERLAP`, `SWAP_ALREADY_STARTED`

**Core rules:**
- There is one `attendances` row per **shift instance** (person × team × work date × shift template). Check-in, leave and absence all write that same row. Every SiAPP view reads from it.
- ⚠ That row is SiAPP's **capture and evidence** (selfie, GPS, QR leader, distance). It is **not** the official record. Every change to it writes a `personel_outbox` event in the same transaction, and Personel receives it asynchronously (07.3). Comment this on the table.
- Check-in is online-only (QR tokens expire in 60 s). Delivery to Personel is not, so a Personel outage never blocks a guard.

---

## Phase 07.1: Schema, today's shifts and selfie check-in/out

**Goal:** A guard sees the shifts they can check into today, and checks in or out with a selfie.

**Build**
- **Table `attendances`** (tenant table, RLS)
  - Shift identity: `id`, `organization_id`, `personnel_id`, `team_id`, `project_id`, `work_date` (DATE), `shift_template_id`, `shift_starts_at`, `shift_ends_at`, `timezone` (snapshot, DL-03).
  - Status: `status` ENUM(`PRESENT`,`LEAVE`,`ABSENT`).
  - Check-in fields: `check_in_at`, `check_in_method` ENUM(`QR`,`SELFIE`,`GROUP`,`FORCE`), `check_in_media_id`, `check_in_latitude`, `check_in_longitude`, `check_in_distance_meters`, `check_in_within_area`, `check_in_qr_danru_id`.
  - Check-out fields: the same columns with the `check_out_` prefix, plus `check_out_update_count` (default 0).
  - Links: `leave_request_id`, `marked_absent_by_id`, `shift_swap_id` (all nullable).
  - Sync: `personel_sync_status` ENUM(`PENDING`,`SYNCED`,`FAILED`) default `PENDING`, `personel_synced_at`.
  - Timestamps.
  - UNIQUE (`organization_id`, `personnel_id`, `team_id`, `work_date`, `shift_template_id`); index (`organization_id`, `team_id`, `work_date`).
  - ⚠ Comment why the shift times and timezone are snapshotted: templates and project settings can be edited later, and history must not change.
- **Table `personel_outbox`** (tenant table, RLS) — the transactional outbox for module 07 (and later 02/05 if needed)
  - `id`, `organization_id`, `event_id` (uuid, UNIQUE; sent to Personel as its idempotency key), `kind` ENUM(`ATTENDANCE`), `aggregate_id` (the attendance id), `payload` (jsonb), `status` ENUM(`PENDING`,`SENT`,`DEAD`), `attempts`, `next_attempt_at`, `last_error`, `created_at`, `sent_at`.
  - Index (`organization_id`, `status`, `next_attempt_at`).
  - `attendance.writeEvent(ctx, q, attendance, type)` (package-private) builds the payload and inserts the row. Every capture usecase calls it inside its transaction. Types: `CHECK_IN`, `CHECK_OUT`, `CHECK_OUT_UPDATED`, `ABSENT`, `ABSENT_CANCELLED`, `CORRECTED`, `GROUP_PRESENT`.
  - Delivery is Phase 07.3. Until then rows just accumulate as `PENDING`.
- **`pkg/platform/geo`**
  - `DistanceMeters(a, b Point) float64` using haversine.
  - Comment the formula and the Earth radius used.
- **Geofence source** ⚠: the team's geofence when `teams.geofence_enabled`, otherwise the project's coordinates and `attendance_radius_meters` (DL-48). Comment the precedence.
- **`attendance.Shifts.Checkable(ctx, personnelID, now)`**
  - Combines `organization.Shifts.Current` with existing records.
  - Each shift gets a `shift_ref`: an opaque base64url of `teamId|workDate|shiftTemplateId`.
- **`GET /attendances/today`** (contract change)
  - Returns `[{ shift_ref, project_name, team_name, shift_name, starts_at, ends_at, status, check_in_at, check_out_at }]`.
- **`POST /attendances/selfie/check-in`** and **`/check-out`** with `{ shift_ref, media_ids: [selfie], latitude, longitude, accuracy_meters }` (`attendance.checkin`; idempotent)
  - The `shift_ref` must be one of the checkable shifts, otherwise `ATTENDANCE_SHIFT_NOT_ACTIVE`.
  - Attach the media with purpose `ATTENDANCE_SELFIE` in the same transaction.
  - ⚠ Geofence: if `distance > radius`, with `strict_attendance_geofence` on return `ATTENDANCE_OUTSIDE_AREA`; otherwise save it with `within_area = false` for the DANRU and admins to see.
  - ⚠ Late check-in: never refused. Store `late_minutes` on the record: whole minutes after `shift_starts_at`, 0 when within the shift's `tolerance_minutes` (past it, counted from the start). Include it in the outbox payload. `teams.reject_attendance_after_tolerance` is no longer read by attendance. Comment it.
  - ⚠ Previous working day: a check-in is refused while the person's latest earlier scheduled non-OFF day is unsettled, as in kpi-personalia-api. The whole day is judged (every record, any shift or regu). Any `PRESENT` without check-out: `ATTENDANCE_PREVIOUS_NOT_CHECKED_OUT`. Otherwise any record settles it: a finished `PRESENT`, an approved `LEAVE`, or an `ABSENT` (a decided day; blocking on it would make every absence decision a lockout). No record at all (alpa): `ATTENDANCE_PREVIOUS_NOT_CHECKED_IN`, but only when the person has a SiAPP check-in for some earlier day: before their first one SiAPP has nothing to judge (go-live, new hires, back-dated schedules). Both 422 with `details.previous_work_date`. Only that one day is looked at, and only check-in is refused (check-out and "update pulang" never are). Comment it.
  - **Check-in:** `INSERT` the row with status `PRESENT`. A unique violation means a check-in already exists: `ATTENDANCE_ALREADY_CHECKED_IN`. An existing `LEAVE` or `ABSENT` row returns `ATTENDANCE_NOT_ALLOWED`.
  - **Check-out:** load the row `FOR UPDATE`. It needs a check-in, otherwise `ATTENDANCE_NOT_CHECKED_IN`.
  - Write the outbox event in the same transaction.
- **`PATCH /attendances/{id}/check-out`** ("update pulang"): overwrites the check-out fields, increments `check_out_update_count`, writes the old values to the audit log and a `CHECK_OUT_UPDATED` event.

**Tests**
- Unit: `DistanceMeters` for 2 known pairs; geofence precedence (team over project).
- HTTP
  - The today list respects the shift window.
  - check-in → check-out → update check-out, each producing one outbox row.
  - A double check-in fails, including two parallel check-ins (and only one outbox row exists).
  - Strict geofence rejects; non-strict geofence flags; a late check-in is accepted and counted in `late_minutes` (0 inside the tolerance).
  - Someone else's media is rejected.
  - An idempotent replay of check-in returns the same body and writes no second outbox row.
  - Cross-tenant: a `shift_ref` built from another organization's team is rejected.

---

## Phase 07.2: QR attendance ⚠

**Goal:** The DANRU shows a rotating QR code; guards scan it to check in or out (§2.6.1, §2.6.5, §3.2.2.1).

**Build**
- **`attendance.QRTokens`** (pure, with an injected clock)
  - Token format: `base64url(payload) + "." + base64url(HMAC-SHA256(payload, ATTENDANCE_QR_SECRET))`.
  - Payload: `{ "v":1, "org", "team", "danru", "dir":"IN"|"OUT", "iat" }`. `team` is the regu **code** and `danru` the personnel id (siapp names regu by code).
  - Valid for `qrTokenTTL = 60s`, plus 5 s clock tolerance. Compare MACs with `hmac.Equal`.
  - ⚠ Comment the threat model:
    - The token is stateless and short-lived, so a shared screenshot stops working after about a minute.
    - Several guards may legitimately scan the same token, because each can check in only once per shift.
    - The organization is inside the signed payload, so a token can never be used in another tenant.
- **`GET /attendances/qr/display?team_code=&direction=IN|OUT`** (contract change: the vendored stub took `team_id` and no direction)
  - Returns `{ token, expires_at }`. Requires `attendance.qr.display` with the regu in the route's resolution, otherwise 403 (also for a regu that does not exist).
  - The app refreshes the token about every 30 s.
- **`POST /attendances/qr/check-in`** and **`/check-out`** with `{ shift_ref, token, latitude?, longitude?, accuracy_meters? }` (`attendance.checkin`; idempotent, `Idempotency-Key` required; contract change: the vendored bodies had `qr_token`, `lat`, `lng`, `photo`, `local_id`)
  - Verify the token: `ATTENDANCE_QR_INVALID` or `ATTENDANCE_QR_EXPIRED`. The token's organization must equal the caller's, and its direction must match the endpoint (a check-out code does not check in): `ATTENDANCE_QR_INVALID`.
  - The token's team must equal the `shift_ref` team, otherwise `ATTENDANCE_QR_WRONG_TEAM`.
  - **Reuse the same private check-in/check-out functions as Phase 07.1**, with the method as a parameter. Do not duplicate them.
  - Store method `QR` and the DANRU's id (also in the outbox payload as `check_in_qr_danru_id` / `check_out_qr_danru_id`). GPS is optional, because the DANRU being physically present is the proof; when a fix is sent it is measured against the project's geofence like a selfie's (strict refuses it).
  - **`ATTENDANCE_QR_SECRET`** (config) empty turns the QR method off: display and scans answer 503 `ATTENDANCE_QR_UNAVAILABLE`, the selfie method keeps working. A set value must be at least 32 bytes (and not a placeholder in production).

**Tests**
- Unit (table-driven): valid token, tampered payload, tampered MAC, expired, wrong secret, another organization.
- HTTP
  - A DANRU gets a token and a guard checks in with it.
  - A member asking for a token gets 403.
  - A token for another team is rejected; an expired token is rejected (fixed clock); another organization's token (same regu code) is rejected; a code of the other direction is rejected; a forged one is rejected.
  - One code serves several guards, and each checks in once. The geofence applies to a fix sent with a scan. Without the secret the QR endpoints answer 503.

---

## Phase 07.3: Delivery to Persist Personel 🔌 ⚠ (built last, after module 16)

> **Priority:** nothing else needs this phase, so it runs after the last module. Until then the `personel_outbox` rows of 07.1 and 07.2 accumulate as `PENDING`; the worker drains them in order when it exists. Do not skip the outbox rows: they are what makes the late delivery lossless.

**Goal:** Every attendance event reaches Personel exactly once in effect, in order per attendance, and failures are visible.

**Build**
- **Periodic task `personel:push`** (asynq, every 15 s; also enqueued right after a capture commits, so delivery is usually immediate).
  - Loops over active organizations; for each, in `WithTenant`:
    1. `SELECT … FROM personel_outbox WHERE status = 'PENDING' AND next_attempt_at <= now() ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED`. ⚠ Comment why `SKIP LOCKED`: several workers can run without sending the same event twice.
    2. ⚠ **Order per attendance:** skip an event when an older event of the same `aggregate_id` is still pending. Comment why: Personel must not receive a check-out before its check-in.
    3. Send with `personel.Attendance.PushEvents`. The payload carries `event_id`, NIK, Personel's project id, `work_date`, shift start and end, the timestamps, method, `within_area`, and a reference to the evidence (no photo bytes).
    4. Per result: accepted or duplicate → `SENT`, and the attendance becomes `SYNCED`; a permanent rejection (4xx) → `DEAD`, attendance `FAILED`, notify the project admins; a transient failure → `attempts + 1`, `next_attempt_at = now + backoff` (30 s, 2 m, 10 m, 1 h, then hourly), `DEAD` after 24 h.
  - The circuit breaker from 00.5 stops the loop early when Personel is down.
- **Admin visibility**
  - `GET /admin/attendances` gains `personel_sync_status` filtering.
  - `POST /admin/attendances/{id}/resync` (contract change): moves the attendance's `DEAD` events back to `PENDING`. Audited.
- **Metrics** (log fields, no new dependency): outbox depth and oldest pending age per organization, logged every run, so an alert can be set on them.

**Tests**
- HTTP + worker inline, against the stub:
  - A check-in reaches the stub once; a replay of the same event id is treated as a duplicate and still marked `SENT`.
  - A check-out queued while the check-in is failing is held until the check-in succeeds.
  - Stub down → retries with backoff; after recovery everything is delivered in order.
  - A 4xx from the stub → `DEAD`, admins notified, resync re-queues it.
  - Two workers running at once never send one event twice.
  - Organization A's events never go out with organization B's credentials or tenant id.

---

## Phase 07.4: My attendance history

**Goal:** The guard's monthly calendar and day detail (§2.6.3.1, §3.2.2.3).

**Build**
- **`GET /attendances/me?month=YYYY-MM`**
  - Returns `[{ work_date, shifts: [{ attendance_id?, shift_name, team_name, status, check_in_at, check_out_at, method }] }]`.
  - ⚠ Merge the scheduled instances (`organization.Shifts.Instances` for the month) with the stored records. Status comes from the record when one exists. Without a record: `MISSING` if the shift has already ended, otherwise `UPCOMING`. Comment that these two are computed, not stored.
  - Comment that this is the field view of SiAPP's captures. HR corrections made directly in Personel are not reflected here; the official totals are Personel's (module 14 reads them back).
- **`GET /attendances/{id}`** — for one record: DANRU name(s), project, team and shift, method and times, distance and within-area flag, selfie views (`media.Service.Views`), `personel_sync_status`. Allowed for the owner, or for anyone whose scope covers the record's team.

**Tests**
- HTTP
  - A month containing present, missing and upcoming days.
  - An overnight shift appears on its start date.
  - The detail includes the selfie URL.
  - Another member's record returns 404; a DANRU of that team can read it.

---

## Phase 07.5: Leave, permit and sick requests (izin, cuti, sakit), owned by SiAPP

> **Decision (replaces D2's "forward to Personel"):** SiAPP owns the whole request: categories, quota, balance, submit, approval and its effect on attendance. **Nothing here calls Personalia**, so there is no seam S5, no mirror and no `personel.Leave`. The rules below copy what `kpi-personalia-api` does today (`internal/modules/requests`, `internal/modules/approval`; "KPI" below), so people get the behavior they already know. Every place where SiAPP differs on purpose is in the deviation table. Where the table or this phase says "KPI quirk", do not copy it.
>
> **What Personel still needs:** the official attendance record (LOCK F5). An approved leave is a day it must not count as absent, so approvals and revocations are written to the outbox like attendance events (step C4). Delivery is 07.3, built last; until then those rows stay `PENDING`. Open decision **D21** covers the Personel side.

**Goal:** A person requests leave, a permit or sick leave. Their DANRU decides it (an admin can stand in or overrule). Quota is checked and reserved the way Personalia does it, and an approved request shows up in attendance (§2.6.4, §2.6.8, §3.2.2.4, §3.2.2.8).

### How Personalia does it (the reference, from KPI)

| Aspect | KPI behavior | Where in KPI |
|---|---|---|
| Kinds | `leave`, `permit`, `sick` (a fourth, `overtime`, exists in the table but is not requestable). Every request is a date range `start..end`, inclusive. | `requests/dto.go:14-64` |
| Approver | Exactly one: the person's `direct_supervisor_id`, frozen in `approver` at submit. No chain. A second tier, HR (by role, scoped to the person's client), may also decide. The owner never decides their own. | `approval.go:117-192` |
| Statuses | `0` waiting, `1` approved by supervisor, `2` approved by HR, `3` rejected, `4` canceled. **1 and 2 both count as approved everywhere**, so the supervisor alone is enough. | `dto.go:69-86` |
| Transitions | Supervisor: `0→1` or `0→3`, nothing once it is `1`. HR: `0→2`, `1→2`, `0→3`, `1→3`. `2`, `3` and `4` are final. Rejecting needs a reason (≤1000 chars); approving ignores one. | `approval.go:125-305` |
| Cancel | Owner only, only while `0`. Two different 409s: already canceled, and not waiting. | `approval.go:429-501` |
| Submit checks, in order | validation → category (exists, right type) → lock the person → approver exists → tenure (annual leave only) → overlap → quota (leave and permit only) → request number → insert → inbox row → ledger deduction. One transaction. | `service.go:512-740` |
| Overlap | Against the person's own requests of **any** type in status `0`, `1`, `2` (not `3`, `4`) whose dates intersect. Ordinary categories conflict with everything. The two "part of day" categories (10, 11) may coexist with each other unless the time windows clash, or there are already two. | `overlap.go` |
| Working days | The person's schedule rows in the range, minus the off shift (`shift_id = 90`), minus holidays for fixed-schedule people. No schedule rows at all in the range → calendar days. A date with no schedule row while others have one → `schedule_missing`. Zero working days → `no_working_days`. | `leaveday.go:74-115` |
| Quota, annual leave (category 1) | A **ledger** (`m_leave_maintenance_detail`). The days are **deducted at submit** and refunded on reject or cancel (idempotent). Never negative. | `ledger.go` |
| Quota, other categories | Per client and category: `quota` minus the number of already-approved requests this year. Nothing is reserved. Sick has no quota. | `quota.go:72-163` |
| Day check | `available < 1` fails; `days > available` fails except for category 3 (maternity). | `service.go:882-892` |
| Number | `LR`, `PR` or `SR` + `YYYYMM` + a 5-digit per-type counter. | `service.go:620-638` |
| Attachment | Optional, except required when sick with `letter_status = true`. | `handler.go:407-409` |
| Effect on attendance | **Nothing is written** when a request is approved: attendance reads approved requests as an overlay. A covering "full day" permit (12) zeroes lateness; "part of day" permits (10, 11) subtract their time from lateness; a day with both clocks stays `Present`. | `attendance/logic.go:182-232`, `history_status.go:224-272` |
| Guards | A missed previous working day is excused by **any** request on it that is not canceled (pending, approved and even rejected). | `attendance/service.go:864-999` |
| Notifications | Submit → the approver (inbox + push + email). Decision → the owner (inbox + push). Cancel → a push to the approver. | `service.go:674-801`, `approval.go:260-303` |
| Lists | Own list: filters `type`, `status`, dates; cursor paging. Approver list: only direct reports, with `overlaps_with` and a coverage figure; detail adds the person's leave balance before and after. | `handler.go:125-237`, `team_requests.go` |

### What SiAPP does differently, on purpose

| # | KPI | SiAPP | Why |
|---|---|---|---|
| 1 | The approver is `direct_supervisor_id` (a person from Personalia). | The approver is the **DANRU of the regu** the request is made for (D2), checked live with `attendance.leave.approve` (TEAM). A DANRU's own request, and a regu with no DANRU that day, go to the admins instead. | SiAPP has regu and DANRU, not supervisors. KPI answers a missing supervisor with a misleading "Employee NIK not found" 404. |
| 2 | HR is a second tier by role and client. | `ORG_ADMIN`, and the `REGION_ADMIN` of the project's region (D18), through `/admin` routes. Same transitions as KPI's HR. | Same shape, SiAPP's roles. |
| 3 | Non-annual quota counts **requests**, but the day check compares **days** to the count left; pending requests count for nothing; the window is `created_at`. | Quota is in **days**. `used` is the working days of requests in status `PENDING` or `APPROVED` whose `start_date` is in the window. Nothing to refund: a rejected or canceled request just stops counting. | KPI quirk: pending requests can all exceed the quota; a day count is compared with a request count. |
| 4 | Annual leave needs 12 months from `join_date`. | No tenure rule. The balance of a person with no grant is 0, so the day check fails the same way. | `join_date` is not in `Identity`. Admins grant the balance (Part A). Add `join_date` to Identity if HR wants the rule itself. |
| 5 | Quota window: calendar year, or the join-date anniversary (client setting). | Calendar year in the project's timezone only. | The anniversary needs `join_date` (see 4). |
| 6 | Holidays are removed from the days of fixed-schedule people. | No holiday table. Working days are the scheduled, non-OFF days; a holiday is an OFF day in the schedule. | SiAPP schedules are explicit per person per date. |
| 7 | A category's special behavior is hardcoded by id (1, 3, 10, 11, 12, 13) and by the name `"Sakit"`. | Behavior is **flags on the category row**. | Ids are not stable across databases, and the name match is fragile. |
| 8 | The ledger writes refunds as `process_id = 3`, which attendance reads as a yearly grant. | Separate ledger kinds: `GRANT`, `DEDUCT`, `REFUND`, `ADJUST`. | KPI quirk (its refund can be counted as entitlement). |
| 9 | A covering rejected request also excuses a missed previous day. | Only `PENDING` and `APPROVED` ones do. | A rejected request is a refusal, not an excuse. |
| 10 | Approved requests are an overlay; no rows are written. | An approved request writes `LEAVE` attendance rows for its leave days (07.1's rule: one row per shift instance, `LEAVE` refuses a check-in). The "part of day" and "on duty" categories write none and only adjust lateness. | 07.1, 07.4, 07.6 and 14 read rows, not requests. Someone on duty elsewhere must still be able to check in. |
| 11 | Category assignment per person is only used to filter the category list; Store ignores it. | No per-person assignment. The list shows the active categories of the type. | The gap is a KPI bug; the per-person list has no owner in SiAPP. |
| 12 | The attachment is a PDF/PNG/JPEG file in multipart. | `media_ids` of one photo (purpose `LEAVE_ATTACHMENT`, 04's image types). The body is JSON. | SiAPP's upload flow is the media module. |
| 13 | The own list defaults to requests covering today. | No default window; newest first. | KPI quirk. |
| 14 | The submit inbox text names the NIK. | It names the person. | KPI quirk. |

### Part A: categories, quota settings and the balance ledger

**Build**
- **Table `leave_categories`** (tenant table, RLS)
  - `id`, `organization_id`, `code` (UNIQUE within the organization), `name` (Indonesian), `type` ENUM(`LEAVE`,`PERMIT`,`SICK`), `active`, timestamps.
  - Flags (⚠ comment that each replaces a KPI hardcoded id): `is_annual` (has the ledger; KPI category 1), `skips_day_check` (KPI 3), `requires_time_range` (KPI 10, 11: needs `start_time` and `end_time`, and is the "part of day" overlap class), `requires_change_day` (KPI 13), `makes_leave_days` (writes `LEAVE` rows; false for KPI 10, 11, 12), `excuses_lateness` (KPI 12: zeroes the day's lateness), `has_quota` (false for sick).
- **Seed** (`seed leave-categories`, run by `EnsureDefaults` for a new organization, like 06.1). The ids are KPI's `m_attendance_leave_permit_categories` from its seeder, which is a recovered snapshot: **confirm them against production before launch**.

  | Code | KPI id | Name | Type | Flags |
  |---|---|---|---|---|
  | `ANNUAL` | 1 | Cuti Tahunan | LEAVE | `is_annual` |
  | `UNPAID` | 2 | Cuti Potong Gaji | LEAVE | |
  | `MATERNITY` | 3 | Cuti Melahirkan (Wanita) | PERMIT | `skips_day_check` |
  | `FAMILY_DEATH_CORE` | 4 | Cuti Keluarga Inti Meninggal | PERMIT | |
  | `FAMILY_DEATH_HOME` | 5 | Cuti Keluarga Serumah Meninggal | PERMIT | |
  | `SPECIAL` | 6 | Cuti Khusus | PERMIT | |
  | `CHILD_BAPTISM` | 7 | Cuti Pembaptisan Anak | PERMIT | |
  | `CHILD_CIRCUMCISION` | 8 | Cuti Penyunatan Anak | PERMIT | |
  | `OTHER_PART_DAY` | 10 | Other Part of Day | PERMIT | `requires_time_range`, not `makes_leave_days` |
  | `ON_DUTY_PART_DAY` | 11 | On Duty Part of Day | PERMIT | `requires_time_range`, not `makes_leave_days` |
  | `ON_DUTY_FULL_DAY` | 12 | On Duty Full Day | PERMIT | `excuses_lateness`, not `makes_leave_days` |
  | `CHANGE_DAY` | 13 | On Change Day (Tukar Hari) | PERMIT | `requires_change_day` |
  | `MARRIAGE` | 15 | Cuti Menikah | PERMIT | |
  | `CHILD_MARRIAGE` | 16 | Cuti Pernikahan Anak | PERMIT | |
  | `CHILD_BIRTH` | 17 | Cuti Kelahiran / Keguguran Anak (Pria) | PERMIT | |
  | `SICK` | 21 | Sakit | SICK | not `has_quota` |
  | `MISCARRIAGE` | 23 | Cuti Keguguran (Wanita) | PERMIT | |

  All others have `makes_leave_days` and `has_quota`; ids 9, 14, 18–20 and 22 are not seeded.
- **Table `leave_category_quotas`**: `category_id`, `client_code` (NULL = the organization's default), `quota_days` (INTEGER ≥ 0), UNIQUE (`category_id`, `client_code`). A client's row wins over the default; **no row means 0**, as in KPI (the category cannot be requested). Seed: no rows, so HR sets them. (`ANNUAL` has none: its limit is the ledger.)
- **Table `leave_client_settings`**: `client_code` PK, `requests_enabled` BOOLEAN. No row means enabled. When false every non-sick request fails `LEAVE_QUOTA_EXCEEDED` (KPI's `can_request_leave_permit = 0`).
- **Table `leave_ledger`** (annual leave only): `id`, `organization_id`, `personnel_id`, `kind` ENUM(`GRANT`,`DEDUCT`,`REFUND`,`ADJUST`), `days` (INTEGER, signed), `balance_after` (INTEGER, CHECK ≥ 0), `leave_request_id` (nullable), `note`, `created_by_id`, `created_at`.
  - ⚠ The balance is the last row's `balance_after`. Write it under the **person's row lock** (the per-person lock that 01's `identity.sql` already documents), not by locking the last ledger row: a first row has nothing to lock. Comment why. Refund is idempotent: a `REFUND` for the same `leave_request_id` already exists → do nothing.
  - UNIQUE (`leave_request_id`) `WHERE kind = 'DEDUCT'` and the same for `REFUND`: a request is deducted once and refunded at most once.
- **Admin endpoints** (`ORG_ADMIN`; a `REGION_ADMIN` for people in their region), all audited (contract changes):
  - `GET|POST /admin/permission-categories`, `GET|PATCH /admin/permission-categories/{code}` (name, flags, `active`; the code, `type` and `is_annual` never change: a PATCH that sends one is 422; categories are deactivated, never deleted, DL-09).
  - `PUT /admin/permission-categories/{code}/quotas` `{ default_days?, clients: [{ client_code, quota_days }] }`; `PUT /admin/leave-client-settings/{client_code}` `{ requests_enabled }`.
  - `GET /admin/personnel/{personnel_id}/leave-ledger` and `POST` `{ kind: GRANT | ADJUST, days, note }` (`days` may be negative only for `ADJUST`; the balance may not go below 0).
- **Field reads**
  - `GET /permission-categories?type=LEAVE|PERMIT|SICK`: active categories, `{ code, name, type, is_annual, skips_day_check, requires_time_range, requires_change_day, makes_leave_days, excuses_lateness, has_quota, … }` (every flag: the app needs them to build the form), with `since` delta sync like the other masters. (Replaces 06.1's proxy.)
  - `GET /permissions/quota?category_code=&team_code=` (`team_code` may be left out when the person serves in one regu; the client whose quota applies is the regu's) → `{ category_code, quota_days, used_days, remaining_days }`. For `ANNUAL`: `quota_days` is the balance, `used_days` is the pending and approved days not yet taken, `remaining_days = balance`. Sick has none (422).
  - `GET /permissions/approver?team_code=` (the same `team_code` rule) → `{ personnel_id, name }` of the regu's DANRU today, or 404 `LEAVE_NO_APPROVER` (the request still works: it goes to the admins; the app shows the line "disetujui admin").

**Tests**
- Seeds are idempotent; a new organization gets all 17 categories with the right flags; another organization's are not visible (RLS).
- Quota resolution: the client's row beats the default; no row → 0; a disabled client → every non-sick request fails.
- Ledger: grant, deduct and refund keep `balance_after` right; a deduction below 0 fails and writes nothing; two parallel deductions of the last days → exactly one succeeds; a second refund of one request writes nothing.
- A category used by a request cannot change its type.

### Part B: submit, list, detail and cancel

**Build**
- **Table `leave_requests`** (tenant table, RLS)
  - `id`, `organization_id`, `number` (UNIQUE within the organization), `personnel_id`, `team_code`, `project_code` (copied: the regu may change later), `type`, `category_id`, `start_date`, `end_date`, `start_time`, `end_time` (nullable `TIME`, local to the project), `working_days` (INTEGER: the days used for quota, `0` for categories without quota), `reason`, `responsibility`, `contact_info`, `change_day_from`, `change_day_to`.
  - Sick only: `hospitalization`, `letter_status` (BOOLEAN), `letter_date`, `disease`. `attachment_media_id` (nullable).
  - Decision: `status` ENUM(`PENDING`,`APPROVED`,`REJECTED`,`CANCELLED`), `approver_personnel_id` (the DANRU at submit, nullable; it only drives the notification, the decision is always checked live), `decided_level` ENUM(`DANRU`,`ADMIN`), `decided_by_id`, `decided_at`, `decision_note`, `cancelled_at`, timestamps.
  - CHECKs: `end_date >= start_date`; the time pair is both or neither; `change_day_to >= change_day_from`.
  - ⚠ Comment that `decided_level` is KPI's `approval_status` 1 vs 2: it is kept because only an `ADMIN` may overrule a `DANRU` approval.
  - Add the foreign key that 07.1 left out: `attendances.leave_request_id → leave_requests(id)` (and an index on it).
- **Table `leave_request_counters`**: (`organization_id`, `type`) → `counter`, incremented in the submit transaction. `number` = `LR` | `PR` | `SR` + `YYYYMM` of the submit date in the project's timezone + the counter as 5 digits. Not reset monthly (as in KPI).
- **Media:** a new purpose `LEAVE_ATTACHMENT` (`ALTER TYPE media_purpose ADD VALUE`; 04's allowed types and size apply; the plan table in 04 lists it).
- **`POST /permissions`** (`attendance.leave.request`; idempotent), body (JSON):
  ```json
  {
    "team_code": "1K-IBMI-S1",
    "type": "LEAVE",
    "category_code": "ANNUAL",
    "start_date": "2026-11-20",
    "end_date": "2026-11-22",
    "start_time": null,
    "end_time": null,
    "reason": "Acara keluarga",
    "responsibility": null,
    "contact_info": null,
    "change_day_from": null,
    "change_day_to": null,
    "hospitalization": null,
    "letter_status": null,
    "letter_date": null,
    "disease": null,
    "media_ids": []
  }
  ```
  - `team_code` may be omitted only when the person serves in exactly one regu on `start_date`; the regu must be one they serve in.
  - **Validation** (422 `VALIDATION_ERROR`, a field error per rule; KPI `handler.go:629-794`): `reason` 1–500; `responsibility`, `contact_info` ≤ 255; `disease` ≤ 500; `type` matches the category's type (`LEAVE_CATEGORY_INVALID`); `end_date >= start_date`; `start_time` and `end_time` come together, and on one day `end_time` is after `start_time`; `requires_time_range` categories need both times; `requires_change_day` categories need both change days and every other category refuses them; `SICK` needs `hospitalization` and `letter_status` and no other type may send the four sick fields; `letter_status = true` needs one `media_ids` entry. A sick request without a category uses `SICK`.
  - **Transaction, in this order** (the order is KPI's; comment it):
    1. Lock the person's row.
    2. Resolve the regu and the approver (the regu's DANRU on `start_date`; none, or the person is that DANRU → the admins' queue).
    3. **Overlap** (below) → `LEAVE_OVERLAP` on `start_date`.
    4. **Working days** (below), then the quota (below) → `LEAVE_SCHEDULE_MISSING`, `LEAVE_NO_WORKING_DAYS`, `LEAVE_QUOTA_EXCEEDED` on `category_code`.
    5. Number, insert, `media.Attach` (owner = the person, purpose `LEAVE_ATTACHMENT`, max 1).
    6. `ANNUAL`: write the `DEDUCT` ledger row. A balance that cannot cover it is `LEAVE_QUOTA_EXCEEDED` and the whole transaction rolls back.
    7. Audit; after commit notify (step C3).
  - **Overlap** (KPI `overlap.go`). Candidates are the person's requests in `PENDING` or `APPROVED` whose dates intersect the new ones, of any type. Normal categories are those without `requires_time_range`; timed ones have it.
    - A new normal request conflicts with any candidate.
    - A new timed request conflicts with any normal candidate.
    - A new timed request conflicts with a timed candidate when their time windows clash on a day both cover (`oldStart < newEnd && oldEnd > newStart`). (KPI checks the windows only when the old range fully contains the new one and calls the rest a legacy gap; SiAPP checks every shared day.)
    - Two or more timed candidates already on the dates conflict with anything (KPI's `shiftCount > 1`).
  - **Working days** (KPI `leaveday.go`): from `organization.Shifts.Instances(start, end)`. No schedule rows at all in the range → calendar days. Otherwise a date in the range with no schedule row → `LEAVE_SCHEDULE_MISSING` (the details name the dates); the days are the scheduled non-OFF dates; zero → `LEAVE_NO_WORKING_DAYS`. `working_days` is stored.
  - **Quota** (sick and `has_quota = false`: skipped):
    - `requests_enabled = false` → `LEAVE_QUOTA_EXCEEDED`.
    - `ANNUAL`: `available` = the ledger balance (grants and refunds already in it, pending deductions already out).
    - Others: `available = quota_days (the client's row, else the default, else 0) − used_days`, where `used_days` = the sum of `working_days` of the person's `PENDING` and `APPROVED` requests of that category with `start_date` in the calendar year of the new `start_date`.
    - Fails when `available < 1`, or when `working_days > available` unless `skips_day_check`.
- **`GET /permissions/me`**: the caller's requests, newest first. Filters `type[]`, `status[]`, `from_date`, `to_date` (intersecting the range, as KPI), `page`, `limit`. Each item: `{ id, number, type, category_code, category_name, status, start_date, end_date, start_time, end_time, working_days, reason, requested_at, decision: { level, by_name, at, note } | null, can_cancel, sick: {…} | null, attachment: <media view> | null }`. The `decision.by_name` is the decider's first name, like KPI's "Disetujui oleh {nama} · {tanggal}".
- **`GET /permissions/{id}`** (contract change): the same item. The owner, or someone whose `attendance.leave.approve` reaches the regu, or an admin in scope; anyone else, and a missing id, get 404 `LEAVE_NOT_FOUND` (never 403).
- **`POST /permissions/{id}/cancel`** (contract change; no body): owner only, only `PENDING`. Another person's → 404. `CANCELLED` already → 409 `LEAVE_ALREADY_CANCELLED`; any other non-pending status → 409 `LEAVE_CANNOT_CANCEL`. Both are checked again after the row lock. `ANNUAL` → `REFUND`. Audit, and notify the approver (step C3).

**Tests** (HTTP, real database)
- Submit: each validation rule above, each with the failing field; a second `POST` with the same `Idempotency-Key` gives the same request and one ledger row.
- Overlap: a pending request blocks a new one on a shared day (any types); a rejected or canceled one does not; two timed requests on one day coexist when the windows do not clash and fail when they do; a normal request over a timed one fails.
- Working days: an OFF day and a weekend OFF are not counted; a schedule gap fails with its dates; a range with no schedule rows counts calendar days; an all-OFF range fails.
- Quota: `ANNUAL` with balance 3 allows 3 days once and then fails; a rejected request refunds; non-annual quota counts the days of pending requests; `MATERNITY` ignores the day check; a disabled client fails; sick always passes.
- Numbers: `LR`, `PR`, `SR` counters are per type and per organization and race-free (parallel submits give different numbers).
- Cancel: only the owner, only while pending; two parallel cancels → one wins, one `LEAVE_ALREADY_CANCELLED`; the annual refund happens once.
- Isolation: another organization's request is a 404; the attachment belongs to the requester.

### Part C: decide, effects on attendance, notifications

**Build**
- **Who decides**
  - The DANRU path: `GET /permissions?team_code=&status=&type=&q=&from_date=&to_date=&page=&limit=`, `POST /permissions/{id}/approve`, `POST /permissions/{id}/decline` (`attendance.leave.approve`, TEAM, checked against the request's `team_code` through `access.Resolve` for today). The list is by the DANRU's teams in scope, **not** by the frozen `approver_personnel_id` (⚠ comment why: KPI's badge counts by the frozen approver while its list uses the live roster, and they disagree after a reassignment).
  - The admin path (contract changes): `GET /admin/permissions` (same filters, `region_code`, `project_code`), `POST /admin/permissions/{id}/approve|decline`. `ORG_ADMIN`, or a `REGION_ADMIN` of the request's project region.
  - Nobody decides their own request (403 `PERMISSION_DENIED`).
  - List items add `person: { personnel_id, name, position }`, `overlaps_with` (other `APPROVED` requests of the same regu's members that intersect, excluding the person: `[{ name, date_range }]`) and `coverage` (members present vs the regu's size on the first overlap day). The detail adds, for `ANNUAL`, `leave_balance: { remaining_days, after_approval_days }` where `after = max(0, balance − working_days)` is computed the way KPI does, plus the attachment's view.
- **Transitions** (KPI `approval.go:125-305`): load `FOR UPDATE`, then
  - DANRU: only from `PENDING` → `APPROVED` or `REJECTED`, `decided_level = DANRU`.
  - Admin: from `PENDING` → `APPROVED` or `REJECTED`; from a DANRU-approved `APPROVED` → `REJECTED` only (KPI's HR rejecting a supervisor-approved request); `decided_level = ADMIN`.
  - Anything else → 409 `LEAVE_ALREADY_DECIDED`. `REJECTED` and `CANCELLED` are final. A reject needs `note` (non-blank, ≤ 1000: 422 on `note`); an approve's note is optional and kept.
  - `ANNUAL` rejected (from either status) → `REFUND`, once.
- **C1. Leave rows on approve** (only when the category `makes_leave_days`): for each scheduled non-OFF shift instance of the person in the range, in every regu they serve in that day, upsert the attendance with `status = LEAVE` and `leave_request_id`, `ON CONFLICT … DO UPDATE … WHERE attendances.status <> 'PRESENT'` (an `ABSENT` becomes `LEAVE`, as KPI's alpa is false under a permit; a `PRESENT` stays). The rows skipped for being `PRESENT` are listed in the response as `skipped`. ⚠ Comment why `PRESENT` wins: the guard already came to work.
- **C2. Reject or cancel after approve** (an admin overruling): delete the `LEAVE` rows with this `leave_request_id` that have no check-in (a row that was somehow filled keeps its evidence and loses only the link); recompute lateness (C5).
- **C3. Notifications** (types in 05; text in Indonesian, with the person's name, not the NIK)
  - Submit → `LEAVE_REQUESTED` to the DANRU, or to the admins of the region when there is none (`ListAdminRecipientsOfRegion`, as for absences).
  - Decision → `LEAVE_DECIDED` to the owner (`decided: approved | rejected`).
  - Cancel → `LEAVE_CANCELLED` (new type) to whoever was notified of the submit.
- **C4. Outbox for Personel** (D21): in the same transaction as the decision, write a `personel_outbox` row, kind `LEAVE` (a new `outbox_kind`; `aggregate_id` = the request id so its events stay ordered), with `{ type: leave.approved | leave.revoked, number, nik, leave_type, category_code, category_name, start_date, end_date, start_time, end_time, working_days }`. `leave.revoked` is sent for a reject or cancel of a request that had been `APPROVED`. ⚠ Comment that it is the only way Personel learns of a leave now. No other state change writes one.
- **C5. Lateness** (KPI `attendance/logic.go:182-213`). The `late_minutes` stored at check-in is recomputed:
  - at check-in, from the person's `APPROVED` requests covering the work date: an `excuses_lateness` category → 0; a `requires_time_range` category → minus the minutes of its window that fall in `[shift start, check-in)`, never below 0;
  - at approve and at C2, for the `PRESENT` rows in the range (the same rule, or the original lateness when nothing covers it any more; keep the original in a column so it can be restored).
- **C6. Guards in 07.1** (`requirePreviousSettled`, replacing the plan text it had): the previous working day is also settled when a `PENDING` or `APPROVED` request of **any category** covers it, **including** a day that is a `PRESENT` row without check-out. (A `LEAVE` row already settles a day.) Not copied from KPI: its 14-day "pending alpa" block and its check-out-without-check-in exception.
- **C7. Views that read leave:** 07.4's calendar already shows `LEAVE` rows. Add `leave_request_id` to the attendance detail (`GET /attendances/{id}`) so the app can open the request.

**Tests** (HTTP, real database)
- DANRU approves: the request is `APPROVED` with `decided_level = DANRU`; the `LEAVE` rows are on the leave days of two regu; a `PRESENT` day is skipped; 07.4's calendar shows `LEAVE`; a guard with an old `ATTENDANCE_PREVIOUS_NOT_CHECKED_IN` can check in while the request is only `PENDING`.
- Transitions: a DANRU cannot decide twice or act after an admin; an admin can reject a DANRU-approved request (rows removed, annual days refunded, lateness restored); a rejected request cannot be re-approved; two parallel approvals → exactly one succeeds; a second DANRU of another regu gets 403; the owner gets 403; a DANRU's own request is not visible to their regu's queue and is decided by an admin.
- Scope: a `REGION_ADMIN` of another region gets 404; the list is by live team scope, not the frozen approver (reassign the DANRU and the new one sees it).
- Effects: a `requires_time_range` or `excuses_lateness` approval writes no `LEAVE` row and the person can still check in; check-in under an approved full-day permit has lateness 0; a part-of-day permit subtracts its window; approving after check-in lowers the stored lateness; revoking restores it.
- Notifications: submit → the DANRU (or the admins when the regu has none); decision → the owner; cancel → the same recipients as submit; the texts name the person.
- Outbox: approve writes one `LEAVE` row, a revoke writes a second, a reject of a `PENDING` request writes none, and the events of one request are ordered.
- Isolation: another organization's request is invisible on every endpoint.

---

## Phase 07.6: Absence (mangkir), group view and admin corrections

**Goal:** The DANRU marks absences and monitors the team; admins correct records (§2.6.6, §2.6.7, §3.2.2.6, §3.2.2.7).

**Build**
- **`POST /attendances/absent`** `{ personnel_id, team_id, work_date, reason }` (`attendance.absence.create`, team in scope; idempotent)
  - `reason` is required (mobile spec ATT §6.3).
  - The target must be a member of the team on that date.
  - In a transaction, for each scheduled instance that date: a `PRESENT` row → `ABSENCE_ALREADY_PRESENT`; otherwise upsert the row as `ABSENT` with `marked_absent_by_id`, and write an `ABSENT` outbox event.
  - No schedule that date: `ATTENDANCE_NO_SHIFT`.
  - Audit. After commit, send `ABSENCE_MARKED`.
- **`DELETE /attendances/absent/{id}`** (contract change; same permission): deletes the mistaken `ABSENT` row and writes an `ABSENT_CANCELLED` event. Audited.
- **Riwayat mangkir:** `GET /attendances?status=ABSENT&month=` (`attendance.absence.view`); a `SelfOnly` resolution returns only the caller's own records.
- **Group view** (`attendance.group.view`, teams in scope) — the manual's Absensi Group (§3.2.2.6):
  - `GET /attendances/group-summary?team_id=&month=` (contract change): per day `{ work_date, scheduled, present, leave, absent, missing, swapped }`.
  - `GET /attendances?team_id=&date_from=&date_to=`: per member and day `[{ personnel, shift_name, status, check_in_at, check_out_at, method, within_area }]`.
  - ⚠ Build the month summary from one `GROUP BY work_date` query over `team_shifts` × `team_members` LEFT JOIN `attendances`. Do not loop per day. Comment the aggregation.
- **Group check-in by the DANRU** — `POST /attendances/group` `{ team_id, shift_ref, personnel_ids[], latitude, longitude }` (contract; persist mobile ATT §6.2)
  - ⚠ Off unless the project setting `allow_group_attendance` is on (add the column in this phase's migration; default false) → otherwise `GROUP_ATTENDANCE_DISABLED`. Comment why: it records presence without each guard's own evidence, which weakens the anti-fraud purpose of QR and selfie (manual §2.6.1); clients that need it must opt in per project.
  - Records method `GROUP` with the DANRU's location, one `GROUP_PRESENT` event per person, audited.
- **Admin corrections** — `GET /admin/attendances` (filters: project, team, date range, status, sync status) and `PATCH /admin/attendances/{id}` `{ status?, check_in_at?, check_out_at?, reason }`
  - `reason` is required; the method becomes `FORCE`; the old values go to the audit log; a `CORRECTED` event goes to Personel.

**Tests**
- HTTP
  - Marking a present guard absent fails; mark then delete works; both produce outbox events.
  - A member sees only their own absences.
  - Group counts match a fixture month.
  - Group check-in is refused while the project toggle is off and works when on.
  - A DANRU cannot view a team outside their projects.
  - An admin correction is audited and queued for Personel.

---

## Phase 07.7: Shift swap (tukar jaga) ⚠

**Goal:** The DANRU assigns a replacement for an absent guard. The replacement then holds an extra shift (§2.6.3.2–3, §2.6.9, §3.2.2.9). As in the manual and the mobile spec, a swap takes effect at once.

**Build**
- **Table `shift_swaps`** (tenant table, RLS)
  - `id`, `organization_id`
  - `replaced_personnel_id`, `replaced_attendance_id`, `replacing_personnel_id`
  - `team_id`, `work_date`, `shift_template_id`, `starts_at`, `ends_at`
  - `note` (nullable), `created_by_id`, `cancelled_at` (nullable), timestamps
  - ⚠ Partial UNIQUE (`organization_id`, `replaced_attendance_id`) `WHERE cancelled_at IS NULL`: one *active* swap per absent record, while keeping cancelled history.
- **Candidate lookups** (`attendance.swap.create`; contract changes)
  - `GET /shift-swaps/replaceable?team_id=&work_date=`: `LEAVE`/`ABSENT` records without an active swap.
  - `GET /shift-swaps/candidates?team_id=&work_date=`: members of the given team who are not on leave or absent that day. Any team in scope, so cross-team and cross-project replacement works.
- **`POST /shift-swaps`** `{ replaced_attendance_id, replacing_personnel_id, note? }` (idempotent). In one transaction, lock the replaced record `FOR UPDATE`, then validate in this order:
  1. Both teams are in scope.
  2. The record is `LEAVE` or `ABSENT`, otherwise `SWAP_REPLACED_NOT_ABSENT`.
  3. The replacing person is not the replaced person, and is not on leave or absent that day.
  4. **No time overlap** with the replacing person's own shift instances that day, otherwise `SWAP_SHIFT_OVERLAP`. Comment the interval rule: back-to-back shifts are allowed.
  5. Insert. A unique violation → `SWAP_ALREADY_EXISTS`.

  Then audit, and after commit send `SWAP_ASSIGNED` to both people.
- **`GET /shift-swaps?status=&date_from=&date_to=`** (contract).
- **`DELETE /shift-swaps/{id}`** (contract change): sets `cancelled_at`; allowed only before the replacing person checks in, otherwise `SWAP_ALREADY_STARTED`.
- **Extend `attendance.Shifts.Checkable`**
  - Active swaps where the person is the replacer are added as extra instances.
  - A check-in on such an instance creates the row with `shift_swap_id` and the **swap's team**, so the QR shown by that team's DANRU validates naturally. The outbox event carries the swap, so Personel can credit the shift to the replacer. Comment this flow.
- **Tabs**
  - `GET /attendances/replaced?month=` ("Digantikan"): `[{ work_date, shift_name, replacing_personnel, replacing_check_in_at, replacing_check_out_at }]`.
  - `GET /attendances/replacing?month=` ("Menggantikan"): the mirror of the above.
- **Extend 07.4 and 07.6:** the calendar includes covered shifts; the group summary counts `swapped`; group rows show `SWAPPED` when the record has an active swap.

**Tests**
- Unit: overlap check for overlapping, back-to-back and overnight shifts.
- HTTP
  - Full flow: leave → approve → swap → the replacer sees 2 shifts today → checks in with the swap team DANRU's QR → both tabs are correct → the stub receives the check-in with the swap reference.
  - Swapping a present guard fails; an overlapping replacer is rejected.
  - Two parallel swaps for the same record: exactly one succeeds.
  - Cancelling after the replacer has checked in fails.
