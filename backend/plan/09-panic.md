# 09: Panic signal

**Purpose:** An emergency alert to the whole team and the project monitor (siren plus push), one-tap access to emergency contacts, a mandatory follow-up report, and resolution by the sender, the DANRU or a web operator.
**Manual refs:** §2.12, §3.2.8, Gambar 12, Tabel 8 (Sinyal Panik rows).
**Needs:** 04, 05, 06, 08.
**Contract operations:** `POST /panics`, `GET /panics`, `POST /panics/{id}/finish`, `GET /admin/panics`, `POST /admin/panics/{id}/resolve`. **Contract change:** `GET /panics/{id}`.
**Error reasons:** `PANIC_NOT_FOUND`, `PANIC_ALREADY_RESOLVED`, `PANIC_NOT_ALLOWED`, `TEAM_REQUIRED`.

**Priority:** This is the most safety-critical feature. The trigger path must be fast and idempotent, with no optional work.

**Lifecycle (DL-31):** `ACTIVE` → `RESOLVED`. The sender or the team's DANRU resolves from mobile by filing the follow-up report (`finish`); a web operator can resolve from the monitor (`resolve`). Exactly one of `resolved_by_personnel_id` / `resolved_by_admin_id` is set.

---

## Phase 09.1: Trigger (offline-capable)

**Goal:** Pressing the panic button alerts the team and the project monitor immediately and returns the emergency contacts.

**Build**
- **Table `panics`** (tenant table, RLS)
  - `id`, `organization_id`, `local_id` (uuid, UNIQUE within the organization), `personnel_id`, `team_id`, `project_id`, `panic_type_id`.
  - `latitude`, `longitude`, `accuracy_meters` (all nullable).
  - `occurred_at`, `occurred_date` (DATE, project timezone), `received_at`.
  - `status` ENUM(`ACTIVE`,`RESOLVED`), `resolved_at`, `resolved_by_personnel_id`, `resolved_by_admin_id` (CHECK: at most one set).
  - Timestamps.
  - Indexes: (`organization_id`, `team_id`, `occurred_at`) and (`organization_id`, `project_id`, `occurred_at`).
- **`POST /panics`** with body `{ local_id, occurred_at, panic_type_id, team_id?, latitude?, longitude?, accuracy_meters? }`. Requires `panic.create`; idempotent.
  - `team_id` may be omitted only when the person has exactly one active membership. Otherwise `TEAM_REQUIRED`.
  - Rate limit: 5 per minute per person.
  - ⚠ When the request is an idempotent replay, return the stored panic and **send nothing**. Comment why: offline retries must not sound the sirens again.
  - ⚠ Persist first, then broadcast (realtime doc §7): an alert must never exist without a record.
- **After commit**
  - `Notify(PANIC_TRIGGERED, Priority: HIGH)` to every team member except the sender, and to the project's admins.
  - `realtime.Publish` `panic.alert` to `team.{teamId}.panic` and `project.{projectId}.panic` with `{ panic_id, panic_type, personnel_ref, latitude, longitude, occurred_at }`. No NIK or full name in the frame.
- **Response:** the panic, plus `emergency_contacts` from `masterdata.ContactsForProject(projectID, panicType.StakeholderTypeID)` for one-tap dialling ("Telp Stakeholder", §3.2.8.1).
- **No media or report fields in this step.** Keep the request path minimal.

**Tests**
- HTTP
  - Team members get an inbox row and a realtime event; the sender does not; a project monitor socket gets it too.
  - A replay creates no second notification.
  - The preferred stakeholder type comes first.
  - A person in several teams who omits `team_id` gets `TEAM_REQUIRED`; a team the person is not in returns 403.
  - Cross-tenant: org B's monitor socket receives nothing.

---

## Phase 09.2: Follow-up report, resolve and history

**Goal:** Close the incident with a report, let operators resolve from the web, and list panic history (§2.12.2, §2.12.3, §3.2.8.1).

**Build**
- **Table `panic_reports`** (tenant table, RLS): `id`, `organization_id`, `panic_id` (UNIQUE), `subject`, `detail` (text), `category_id` (nullable, REPORT kind), `created_by_id`, `created_at`. Attachments use media purpose `PANIC_REPORT`.
- **`POST /panics/{id}/finish`** `{ subject, detail, category_id?, media_ids }` (offline-capable)
  - Allowed for the sender (`panic.finish` SELF) or the team's DANRU (`panic.finish` TEAM), otherwise `PANIC_NOT_ALLOWED`.
  - In one transaction: lock the panic `FOR UPDATE`; already resolved → `PANIC_ALREADY_RESOLVED`; insert the report; attach media; set `RESOLVED`, `resolved_at`, `resolved_by_personnel_id`.
  - ⚠ The mobile spec makes the report mandatory and keeps the panic `ACTIVE` until it is filed. Comment it.
  - After commit: `realtime.Publish` `panic.resolved` to both rooms, and `PANIC_RESOLVED` to the team. ⚠ Comment why: other devices stop the siren when they receive it.
- **`POST /admin/panics/{id}/resolve`** `{ note }` (admin with the project in scope): sets `RESOLVED` with `resolved_by_admin_id`, audited, same broadcast. The follow-up report can still be filed afterwards by the sender (the `finish` call then only adds the report).
- **History**
  - `GET /panics?scope=mine|team|project&team_id=&project_id=&month=&panic_type_id=` (`panic.view`, filtered by scope). "Sinyal Panik Saya" is `mine`; "Group" is `team`; the DANRU's per-project view (Tabel 8, missing from the mobile spec) is `project`.
  - `GET /panics/{id}` (contract change): sender, type, location, report and media views; allowed for the sender or anyone with `panic.view` scope over the team.
  - `GET /admin/panics?project_id=&status=&date_from=&date_to=` for the web monitor.

**Tests**
- HTTP
  - Finish → resolved → `panic.resolved` is emitted on both rooms.
  - A member who is not the sender cannot finish; the team's DANRU can.
  - A second finish fails; an admin resolve followed by the sender's report works.
  - A member sees their own team's history but not other teams'; a DANRU sees the whole project.
