# 16: Cleaning vertical

**Purpose:** The two features that only cleaning projects use: **E-Memo** (memo threads) and **Generate QR** (printable QR codes for cleaning zones). Cleaning is **not** a separate product: it is the same backend with `projects.business_line = cleaning` (LOCK/03 §2, DL-20). This module adds only what is cleaning-specific; attendance, patrol routes, reports and panic for cleaning projects use the normal modules, and only the labels differ.

**Manual refs:** none (new in PERSIST). Specs: `persist-ecosystem/docs/02-web-dashboard/12-cleaning-vertical.md` (WEB-34, WEB-35), `03-mobile-app/00-README.md` (MOB-14, MOB-15), `04-backend-core/01-data-model.md` §11, GAP-018.

**Needs** (per phase, because 16.1/16.2 do not need patrol or analytics):
- 16.1: 01, 02.1, 02.2, 03 (03.1 and 03.2), 04.2, 05.1.
- 16.2: 16.1.
- 16.3: 01.3 (`access.RequireAdmin`), 02.1, 04.1 (`media.Storage`), 10.1 (zones and the QR renderer). **Not 14.2**: see "PDF library" in 16.3. This is why E-Memo can ship before modules 10 and 14.

**Contract operations** (all exist in the contract today; shapes below are the contract's, verified against `api/openapi.yaml`):
- E-Memo: `GET|POST /e-memos`, `GET|PATCH|DELETE /e-memos/{id}`, `POST /e-memos/{id}/responses`.
- QR: `GET /qr-codes`, `POST /qr-codes/generate`.
- **Contract changes** (one PR to `persist-ecosystem` in Phase 16.1, linked in its commit body):
  1. `POST /e-memos/{id}/read` (new): marks the memo read for the caller. Reading must not be a side effect of `GET`.
  2. `EMemoCreate.media_ids: int64[]` (optional, max 5) and `EMemo.photos` (the media views from 04.2). Replies take no attachments.
  3. `maxLength`: `EMemoCreate.body` 10000, `EMemoResponseCreate.body` 5000.
  4. `QrGenerateRequest.layout` becomes an enum: `grid-2x4` (default), `grid-3x5`, `single`.
  5. Descriptions only (no schema change), see 16.3: `QrCode.qr_token` is the exact string the QR encodes; for a bulk request every `QrCode` carries the same `content_url`; `revoked` is always `false` in the current design; `generated_at` is the last rotation time.
  6. The `info` header says E-Memo and Generate-QR are "out of scope for MVP / Phase 2". DL-59 cancelled that split: remove the sentence.

**Error reasons** (reason goes in `error.details.reason`; `error.code` is the contract's `ErrorCode`):

| Reason | `error.code` | HTTP | When |
|---|---|---|---|
| `FEATURE_NOT_ENABLED` | `FORBIDDEN` | 403 | The caller has no cleaning project in scope, or the target project is not a cleaning project. Declared in 03.2; add it to 03's error-reasons line. |
| `EMEMO_NOT_FOUND` | `NOT_FOUND` | 404 | The memo does not exist, is deleted, is in another organization, or the caller is neither a participant nor an admin in scope. |
| `EMEMO_NOT_SENDER` | `FORBIDDEN` | 403 | `PATCH` or `DELETE` by someone who can see the memo but did not send it. |
| `EMEMO_NO_RECIPIENTS` | `UNPROCESSABLE` | 422 | The recipients resolve to nobody (after removing the sender). |
| `EMEMO_TOO_MANY_RECIPIENTS` | `UNPROCESSABLE` | 422 | More than 1000 resolved recipients. |
| `QR_NO_ZONES` | `UNPROCESSABLE` | 422 | A bulk request resolves to no zone the caller may print. |
| `QR_TOO_MANY_ZONES` | `UNPROCESSABLE` | 422 | More than 200 zones in one request. |

Existing reasons reused: `PERMISSION_DENIED`, `NOT_ADMIN` (03, 01.3), `PROJECT_NOT_FOUND` (02), `ZONE_NOT_FOUND` (10), `IDEMPOTENCY_KEY_REUSED` (00.4), and the new `IDEMPOTENCY_RESPONSE_NOT_STORED` (00.4: a 2xx whose response could not be stored; see `pkg/platform/idem`).

---

## Decisions made in this plan

The sources disagree in places. Where they did, this is what was chosen and why. **Changing one of these changes the tables, so settle it before starting 16.1.**

| # | Question | Decision | Why |
|---|---|---|---|
| D16-1 | Plan said `team_id?` + `media_ids`; contract says `recipients: { team_ids[], personnel_ids[] }` | **Follow the contract.** Audience is stored as a resolved snapshot (`e_memo_recipients`) plus the requested ids. `media_ids` is added to the contract (change 2). | The contract wins over this plan (README, "Sources, in order of authority"). The web spec and the data model also describe team and individual recipients. |
| D16-2 | Plan said soft delete; the contract and web spec say DELETE cascades to responses | **Soft delete** (`deleted_at`). The FK `ON DELETE CASCADE` stays, so a hard purge (organization erasure) still removes responses. | Memos are internal records worth keeping for audit; the contract text "CASCADE" describes the FK and is still true. The memo disappears for everyone, as the web spec wants. |
| D16-3 | Who may use `/e-memos`: the field permission `ememo.use`, or web admins too? | **Either.** A caller needs the field grant `ememo.use` **or** a web role assignment (`ORG_ADMIN`, or `PROJECT_ADMIN` for their `project_ids`). Admins have a personnel row, so they send and reply as themselves. | The web spec lets admins compose, view and reply; the contract has one set of routes (no `/admin/e-memos`). |
| D16-4 | Where the cleaning gate runs | **In the usecase**, through one function, not only as route middleware. | `RequireBusinessLine` as middleware needs a project in the request; here the project comes from the memo, the recipients, or a list of zone ids. |
| D16-5 | Flag mechanism | **`projects.business_line` is the only switch.** The docs' `e_memo`/`generate_qr` flags are both "ON for cleaning" and are derived, not stored. The web docs' per-organization `organizations.enabled_business_lines` is **not built** (README: "The backend never forks a module for it"). | One mechanism, no second place to get out of sync. If a customer needs a per-organization off switch later, it is an additive `AND` in the same function. |
| D16-6 | `e_memos.business_line` column | **Keep it**, as `business_line NOT NULL DEFAULT 'cleaning' CHECK (business_line = 'cleaning')`. | The web spec (AC-W34-4) requires the CHECK and an index on it; data model §11 omits the column, so fix §11. It is defence in depth only: the real gate is D16-4. |
| D16-7 | The `PERSONNEL` identity QR in the old plan (`PERSIST-PERSON:1:…:{hmac}`, `QR_IDENTITY_SECRET`) | **Dropped from 16.** | The contract (`QrCode`, `QrGenerateRequest`), the web spec and MOB-15 all say "QR identitas **zona**" (a zone's identity). No document says who would scan a personnel card or what it would open, and nothing in modules 01 to 15 verifies one. See open question Q1. |
| D16-8 | Who may print QR codes | **Admins only** (`ORG_ADMIN`; `PROJECT_ADMIN` for their projects), by mounting `access.RequireAdmin` on the two routes individually. The field catalog has no QR permission. | The web spec limits it to admins. A printed code contains the zone secret (10.1), and rotating it invalidates stickers. See open question Q2 for mobile. |
| D16-9 | Revoke vs regenerate | **Only rotate** (`POST /admin/zones/{id}/rotate-qr`, built in 10.1). It revokes the old secret and issues a new one in one step. There is no revoke-without-reissue. | The contract has no revoke endpoint; a zone with no valid QR helps nobody. Scanning an old code already fails with `ZONE_QR_INVALID` (10.2). Open question Q3. |
| D16-10 | Memo timestamps for offline sends | `created_at` = the time the server received it; `synced_at` = the same instant. No `occurred_at`. | The contract has no `occurred_at` on memos, and a memo is not evidence. So the offline helper `OfflineCreate.Check` (which requires `occurred_at`) is **not** used: 16.1 adds `idem.CheckLocalID`. |

---

## Rules shared by every phase in this module

- ⚠ **The cleaning gate** is `access.CheckBusinessLine(ctx, line, projectIDs...)`, in `pkg/access`. 03.2's `RequireBusinessLine` middleware calls it; 16.1 exports it if 03.2 did not. Comment that `business_line` is a dimension **inside** a tenant and never a bypass of `organization_id` isolation: RLS still applies to every query, and the gate only decides whether a cleaning feature is available at a project.
- **Error order for a project id in a query or body** (decided once, tested once in 16.1):
  1. not visible (another organization, or no such project) → 404 `PROJECT_NOT_FOUND`;
  2. visible but outside the caller's scope → 403 `PERMISSION_DENIED`;
  3. in scope but `business_line <> cleaning` → 403 `FEATURE_NOT_ENABLED`.
  A caller who has no cleaning project in scope at all gets `FEATURE_NOT_ENABLED` on every E-Memo route, even without a `project_id`.
- **The caller's scope** is computed once per request by `ememo.Actor`: `{ PersonnelID, AdminAll bool, AdminProjectIDs, FieldTeamIDs, CleaningProjectIDs }`.
  - Admin part: from `access.AdminScope` (01.3).
  - Field part: `access.Resolve(ctx, personnelID, "ememo.use", today)` (03.2), with `TeamIDs` **filtered to teams whose project is cleaning**. ⚠ Comment why: the grant is per organization and team role, not per business line (03.1 only hides it from `/auth/me`), so a person who is in both a security and a cleaning team must not be able to memo the security team.
  - `CleaningProjectIDs` = the projects reachable through either part, with `business_line = cleaning` and `active`.
  - No admin role and no `ememo.use` grant → 403 `PERMISSION_DENIED`. Empty `CleaningProjectIDs` → 403 `FEATURE_NOT_ENABLED`.
- **Idempotency.** `POST /e-memos`, `POST /e-memos/{id}/responses` and `POST /qr-codes/generate` go through `idem.Middleware(required: true)` (00.4). A replay returns the stored response and runs nothing: no insert, no notification, no storage write.
- **Business dates and timezones.** "Today" for membership checks is the business date in the memo's project timezone, or `Asia/Jakarta` when the memo has no project (DL-03). Date filters on lists use the same rule (never a UTC day).
- **Limits.** Subject 255 characters (contract), memo body 10000, reply 5000, 5 attachments, 1000 recipients, 200 zones per QR request.
- **Logs and audit never contain** a QR payload or `qr_secret` (README, Security).

---

## Phase 16.1: E-Memo data model and compose

**Goal:** A cleaning person or admin can send a memo to teams and people in their scope, with optional attachments, and recipients are notified. Nothing about reading or replying yet.

**Build**
- **Migration `e_memos`** (tenant table, RLS, README conventions):
  - `id`, `organization_id`, `sender_personnel_id` (FK `personnels`)
  - `project_id` (FK `projects`, nullable: an organization-level memo, web spec edge case 6)
  - `business_line` ENUM (the type from 02.1) `NOT NULL DEFAULT 'cleaning' CHECK (business_line = 'cleaning')` (D16-6)
  - `subject` VARCHAR(255) `CHECK (btrim(subject) <> '')`, `body` TEXT (nullable)
  - `recipient_team_ids` BIGINT[] and `recipient_personnel_ids` BIGINT[] `NOT NULL DEFAULT '{}'`: the audience **as requested**, so the API can echo `recipients` exactly. ⚠ Comment: these are an echo, validated at write time, not a source of truth for who received the memo (that is `e_memo_recipients`); arrays have no FK on purpose.
  - `local_id` UUID (nullable), `synced_at` timestamptz, `created_at`, `updated_at`, `deleted_at`
  - ⚠ `UNIQUE (organization_id, sender_personnel_id, local_id) WHERE local_id IS NOT NULL`. Comment why the sender is in the key: a `local_id` that is unique only per organization would let user B send user A's `local_id` and be handed A's memo on the "duplicate" path. This index is also the replay guard after the idempotency key has expired (TTL 8 days, 00.4).
  - Indexes: (`organization_id`, `sender_personnel_id`, `created_at` DESC) and (`organization_id`, `project_id`, `created_at` DESC), both `WHERE deleted_at IS NULL`; `pg_trgm` GIN on `subject` and on `body` (needs the extension from 02/11; create it with `IF NOT EXISTS`).
- **Migration `e_memo_recipients`** (tenant table, RLS): `id`, `organization_id`, `e_memo_id` (FK, `ON DELETE CASCADE`), `personnel_id` (FK), `read_at` (nullable), `created_at`, `updated_at`.
  - `UNIQUE (organization_id, e_memo_id, personnel_id)`; index (`organization_id`, `personnel_id`, `e_memo_id`) for the inbox.
  - ⚠ Comment why it is a **snapshot**, as `instruction_recipients` is (12.1): a person who joins or leaves a team later must not gain or lose a memo already sent. Read status lives here, so `e_memos` has no mutable read column (AC-W34-12).
- **Migration `e_memo_responses`** (tenant table, RLS): `id`, `organization_id`, `e_memo_id` (FK, `ON DELETE CASCADE`), `personnel_id` (FK), `body` TEXT `NOT NULL CHECK (btrim(body) <> '')`, `local_id` UUID (nullable), `created_at`, `updated_at` (never changes: kept for the README convention).
  - `UNIQUE (organization_id, personnel_id, local_id) WHERE local_id IS NOT NULL` (same reasoning as above).
  - Index (`organization_id`, `e_memo_id`, `created_at`, `id`) for the thread.
- **`idem.CheckLocalID(r, localID string) error`** in `pkg/platform/idem`: the "local_id must equal the `Idempotency-Key` header, compared as UUIDs" half of `OfflineCreate.Check`, which now calls it. Unit-tested. (D16-10)
- **`ememo.Actor`** and **`access.CheckBusinessLine`** as described in the shared rules.
- **`ememo.resolveRecipients`**: a **pure** function over loaded data (table-driven tests). ⚠ Comment each step:
  1. At least one of `team_ids`, `personnel_ids`, otherwise `VALIDATION_ERROR` on `recipients`.
  2. Every team must exist (RLS hides other organizations' teams, which therefore look missing → `VALIDATION_ERROR`, field `recipients.team_ids`, code `NOT_FOUND`), be active and not deleted.
  3. When the request has a `project_id`, every team and every person must belong to **that** project (web spec AC-W34-5), otherwise `VALIDATION_ERROR` code `NOT_IN_PROJECT`. This is checked **before** scope, so a typo is a 422, not a 403.
  4. Every team must be in the caller's scope: for a field caller, in `FieldTeamIDs`; for an admin, in a project of `AdminProjectIDs` (or any project for `AdminAll`). Otherwise 403 `PERMISSION_DENIED`. Every team's project must be a cleaning project, otherwise 403 `FEATURE_NOT_ENABLED`.
  5. Expand each team to its members **as of today** (`organization.Query.TeamMembers`, 02.2).
  6. Add `personnel_ids`: each must be an active member, as of today, of a team that passes step 4. Otherwise 403 `PERMISSION_DENIED`. Do not reveal whether the person exists elsewhere.
  7. Remove the sender, dedupe.
  8. Empty → `EMEMO_NO_RECIPIENTS`. More than 1000 → `EMEMO_TOO_MANY_RECIPIENTS`.
- **`POST /e-memos`** (`idem` required; an *offline-capable* create):
  - **Body** (contract): `{ local_id?, project_id?, subject, body?, recipients: { team_ids?, personnel_ids? }, media_ids? }`. `organization_id`, `business_line` and `sender_personnel_id` come from the context, **never** from the body.
  - Order: decode and validate (`subject` not blank, lengths) → `idem.CheckLocalID` when `local_id` is present → `Actor` → project check (error order above) → `resolveRecipients`.
  - One transaction (`WithTenant`):
    1. insert `e_memos` (`synced_at = now()`); on a unique violation of the `local_id` index, load the existing memo **of the same sender** and return it as a replay (200, no side effects);
    2. insert the recipients with `:copyfrom`;
    3. `media.Service.Attach(ctx, q, AttachInput{ MediaIDs, OwnerID: sender, Purpose: EMEMO, EntityType: "E_MEMO", EntityID: id, Max: 5 })`;
    4. audit `ememo.create` (memo id, project id, recipient count; never the body).
  - Response: 201 with `EMemo` (`recipients` echoed, `is_read: null` for the sender, `photos` from `media.Service.Views`). A replay answers 200.
  - **After commit:** `notifications.Service.Notify` with `EMEMO_CREATED` to every recipient, `ExcludePersonnelID` = sender, entity `E_MEMO`, data `{ memo_id, project_id }`. Skipped when `idem.Replayed` or when the insert was a `local_id` replay.
- **Notification type.** Migration `ALTER TYPE notification_type ADD VALUE 'EMEMO_CREATED'` (and `'EMEMO_REPLIED'`, used in 16.2), in a goose file marked `-- +goose NO TRANSACTION` (an enum value added inside a transaction cannot be used until it commits). Add both to `Render` in `pkg/notifications/templates.go` with Indonesian text; the `exhaustive` linter then forces every type to have a case. Labels follow the project's `business_line` alias table (05.1). Body: `"{sender} mengirim memo: {subject}"`, subject cut at 80 characters.
- **Contract PR** (changes 1 to 6 above). `make contract` and the generated models must be committed.

**Tests**
- **Unit**
  - `resolveRecipients`: a team expands to members; a person in two teams is deduped; the sender is removed; sender alone → `EMEMO_NO_RECIPIENTS`; 1001 recipients; a team of another project when `project_id` is set; an out-of-scope team; an ended membership is ignored.
  - `idem.CheckLocalID`: equal (any letter case), different, malformed.
  - Actor: an `ORG_ADMIN`; a `PROJECT_ADMIN` of one project; a DANRU whose `FieldTeamIDs` mix a security and a cleaning team → only the cleaning team survives.
- **HTTP**
  - Happy path: a DANRU of a cleaning team sends to their team; every other member has an `e_memo_recipients` row and one `EMEMO_CREATED` notification; the sender has none.
  - An admin with no team membership sends to a team in their project.
  - Error matrix: a security project → `FEATURE_NOT_ENABLED`; a project of another organization → 404 `PROJECT_NOT_FOUND`; a team outside scope → `PERMISSION_DENIED`; blank subject → `VALIDATION_ERROR`; no recipients; a person with neither grant nor admin role → `PERMISSION_DENIED`.
  - A person with only security teams gets `FEATURE_NOT_ENABLED`; `/auth/me` does not list `ememo.use` for them (03.1).
  - Memo without `project_id`: allowed when every recipient resolves to cleaning teams; refused when one does not.
  - Idempotency: the same key and body → 200, one row, one set of notifications; the same key with another body → 409; after deleting the idempotency row, the same `local_id` from the same sender still returns the first memo, and the same `local_id` from **another** sender creates a separate memo and never returns the first one.
  - `media_ids`: attached with purpose `EMEMO`; another person's media → `MEDIA_NOT_FOUND`/mismatch error from 04.2; the whole create rolls back.
  - DB: inserting an `e_memos` row with `business_line = 'security'` fails the CHECK.
  - Cross-tenant: a token from another organization cannot use this organization's team ids (→ validation `NOT_FOUND`), and RLS isolation holds on all three tables (the README test covers `FORCE` and the policy).

---

## Phase 16.2: Inbox, thread, replies, read, edit and delete

**Goal:** Participants read their memos, reply in a thread, mark them read; the sender edits or deletes.

**Build**
- **Who can see a memo** (`ememo.canSee`): the sender, a recipient (snapshot), or an admin whose scope covers the memo (`AdminAll`, or `project_id` in `AdminProjectIDs`; a memo with no project is visible to `AdminAll` only). Anyone else, and any memo that is deleted or in another organization → 404 `EMEMO_NOT_FOUND`. ⚠ Comment why 404 and not 403: a 403 would confirm that the id exists.
- **`GET /e-memos`** (contract params: `page`, `limit`, `sort`, `q`, `project_id`, `date_from`, `date_to`, `box`):
  - `box=inbox`: memos where the caller has a recipient row. `box=sent`: `sender_personnel_id = caller`. No `box`: for a field user, inbox ∪ sent; for an admin, inbox ∪ sent ∪ every memo in their admin scope (web spec §2.2: admins see all memos of their projects).
  - `q`: `ILIKE '%' || q || '%'` on `subject` and `body`, escaping `%` and `_` (README, Search).
  - `date_from`/`date_to`: on `created_at` converted with the memo's project timezone (`coalesce(projects.timezone, 'Asia/Jakarta')`), inclusive of the whole `date_to` day.
  - `sort`: `created_at` only, default descending; add `id` as the tie-breaker.
  - Item = `EMemo` without `responses`; `is_read` is `read_at IS NOT NULL` for the caller as a recipient, and `null` when the caller is not a recipient. One query with a `LEFT JOIN e_memo_recipients` on the caller; the count uses the same filters.
  - `ListEnvelope` with `pagination` (small table, so `COUNT(*)` is fine).
- **`GET /e-memos/{id}`**: `EMemo` with `responses` ordered `created_at ASC, id ASC` and `photos`. Read-only: it does **not** mark the memo read.
- **`POST /e-memos/{id}/read`** (contract change 1): sets `read_at = coalesce(read_at, now())` for the caller's recipient row. A caller who is not a recipient (sender, admin) gets 200 and no change. Idempotent by nature; no `Idempotency-Key`.
- **`POST /e-memos/{id}/responses`** (`idem` required; *offline-capable*):
  - Body `{ local_id?, body }`, `body` not blank, at most 5000 characters. `idem.CheckLocalID` as in 16.1.
  - Allowed for anyone who `canSee` the memo and still passes the `Actor` check. ⚠ In the transaction first `SELECT … FROM e_memos WHERE id = $1 AND deleted_at IS NULL FOR SHARE`. Comment why: a reply that races with the sender deleting the memo must either land before the delete or fail with 404, never attach to a deleted memo.
  - Insert the response; on a unique violation of the `local_id` index load the caller's existing response and return it (200, no side effects).
  - Response 201 `EMemoResponse`.
  - **After commit** `EMEMO_REPLIED` to the memo's sender plus everyone who replied earlier, minus the author, deduped; entity `E_MEMO`, data `{ memo_id }`. Not sent for a replay. Recipients who have never replied are not notified (this is the existing rule of this module).
- **`PATCH /e-memos/{id}`** (`EMemoUpdate`: `subject`, `body` only; recipients and project never change): sender only. Visible but not the sender → 403 `EMEMO_NOT_SENDER`. Audit `ememo.update` with before and after of the changed fields. No notification. Last write wins (no ETag in the contract); say so in the comment.
- **`DELETE /e-memos/{id}`**: sender only (same errors). Sets `deleted_at`; recipients and responses stay in the tables (D16-2). Audit `ememo.delete`. Answers 200 `SuccessEnvelope`. Deleting an already deleted memo is 404.
- Replies are **not** audited (high volume, and they are not edits of a record).

**Tests**
- **HTTP**
  - Thread: create → two recipients reply, then the sender → `GET` returns the replies in order; notifications: the sender is told of reply 1, the sender and the first replier are told of reply 2, the author never is.
  - Inbox, sent and the default box for a field user and for an admin; `q` finds a subject, a body, and treats `%` literally; the date filter uses the project timezone (a memo created at 17:30 UTC lands on the next Jakarta day); pagination and `sort`.
  - Visibility: an unrelated team member gets 404; a `PROJECT_ADMIN` of another project gets 404; an `ORG_ADMIN` sees it; a memo without a project is visible to `ORG_ADMIN` only.
  - Read: `POST …/read` flips `is_read` for the recipient only; `GET` alone does not; the sender gets 200 and `is_read` stays `null`.
  - Reply validation: blank body → `VALIDATION_ERROR`; replying to a deleted memo → 404; two parallel replies both persist, ordered.
  - Idempotency: a replayed reply sends no second notification and creates no second row; the `local_id` rules from 16.1 hold for responses.
  - Edit and delete: the sender succeeds; a recipient gets 403 `EMEMO_NOT_SENDER`; an unrelated person gets 404; after delete, `GET`, the lists and replies all give 404/empty, while the rows remain.
  - Race: a reply and a delete started together end in "reply stored, then deleted" or "reply 404", never a reply on a deleted memo (run with `-race`).
  - Cross-tenant: org B's token gets 404 for every org A memo id on every route.

---

## Phase 16.3: Generate QR

**Goal:** Admins print the QR codes of their cleaning zones: one zone or many, as PNG, SVG or a print-ready PDF. Rotating a code reuses 10.1.

**Build**
- **One code path for the QR itself.** 10.1 owns the payload `PERSIST-ZONE:1:{organizationId}:{zoneId}:{qrSecret}` and the renderer. ⚠ Comment that Generate QR reuses them so a printed cleaning code and a security code are the same format and the scan in 10.2 verifies both. They live in `pkg/zoneqr` (`Payload(org, zoneID, secret)`, `RenderPNG`, `RenderSVG`). If 10.1 was built with them inline, **move them there first** in this phase; do not copy.
- **Column.** `patrol_zones.qr_rotated_at timestamptz NOT NULL DEFAULT now()`, set at creation and by `rotate-qr`. 10.1 should include it; if it does not, add it here with `ADD COLUMN IF NOT EXISTS`. It is `QrCode.generated_at`.
- **Storage.** Add `Put(ctx, key, contentType string, body io.Reader) error` to the `media.Storage` interface (04.1), in both drivers (`S3Storage` and `LocalStorage`, written atomically like the local blob upload), and to the storage conformance suite. Artifacts go under `{organization_id}/qr/{yyyy}/{mm}/{uuid}.{ext}`.
  - ⚠ Comment the retention: a generated file contains the zone secrets, so the key is random and the URL is presigned (24 hours). Objects expire after 2 days. Why 24 h and 2 days: an idempotent replay returns the stored response, whose URL must still work for as long as a client can reasonably retry; after that the client generates again with a new key.
  - **How they expire depends on `STORAGE_DRIVER`:**
    - `minio` / `s3`: a **bucket lifecycle rule** on the `*/qr/` prefix. Add it where the bucket is created: the MinIO init in docker-compose, and the deployment notes.
    - `local`: there is no lifecycle, so `media:cleanup` (04.2) also deletes files under `*/qr/` older than 2 days.
    - Test both drivers.
- **PDF library.** `github.com/go-pdf/fpdf` is added by **whichever of 14.2 and 16.3 lands first**; the other reuses it. State it in the commit body. This is what removes the dependency on 14.2 (which needs 07, 10 and 11).
- **`GET /qr-codes?project_id=&zone_id=&page=&limit=&sort=`** (`access.RequireAdmin` on the route, D16-8): the registry of printable zones, one `QrCode` per zone, from `patrol_zones` of **cleaning** projects within the caller's admin scope.
  - `zone_id`, `project_id` follow the error order of the shared rules.
  - Fields: `zone_id`, `organization_id`, `project_id`, `zone_name` (alias if set, else name), `qr_token` (= the exact payload string the QR encodes), `revoked: false`, `generated_at = qr_rotated_at`. No `format` and no `content_url` (nothing generated yet).
  - ⚠ The response carries the zone secret, which is why this route is admin-only and the field routes of 10.1 never return it. Comment this; add a test that the field `GET /zones` still has no secret.
  - Only active zones are listed, unless the request names one `zone_id` (then that zone, active or not).
- **`POST /qr-codes/generate`** (`access.RequireAdmin`, `idem` required). Body (contract `QrGenerateRequest`): `{ zone_id? | zone_ids? | project_id?, format, layout?, include_inactive? }`.
  - **Selection:** exactly one of `zone_id`, `zone_ids`, `project_id` (else `VALIDATION_ERROR`).
    - `zone_id`: a zone that is not visible → 404 `ZONE_NOT_FOUND`; outside admin scope → 403 `PERMISSION_DENIED`; in a non-cleaning project → 403 `FEATURE_NOT_ENABLED`.
    - `zone_ids`: ids that are not visible, out of scope or not in a cleaning project are **dropped silently** (the contract says so; this also makes another organization's ids harmless). Nothing left → 422 `QR_NO_ZONES`.
    - `project_id`: all active zones of that project (inactive too with `include_inactive`), error order as in the shared rules; none → `QR_NO_ZONES`.
    - More than 200 zones → `QR_TOO_MANY_ZONES`.
  - **Formats:**
    - `png`: 800×800 px per code, the code only (a sticker printer adds its own text; the labelled sheet is the PDF). One zone → the `.png`; several → a **ZIP** of PNGs.
    - `svg`: same, vector. Several → a ZIP.
    - `pdf`: always **one PDF**, A4 portrait, label under each code (zone alias or name, cut to two lines, plus the project name in small type). Layouts: `grid-2x4` (default, 8 per page), `grid-3x5` (15 per page), `single` (one large code per page). Page footer: project name and page `n/N`.
    - ⚠ fpdf's built-in fonts are cp1252 only: pass every label through the fpdf unicode translator so a name such as "Lantai 2 – Toilet" or "é" cannot corrupt the page. Comment it.
    - ZIP entry names are `{project_code}-{zone-name-slug}-{zone_id}.{ext}`; the id guarantees uniqueness.
  - Render in memory (200 codes is small), write one object with `Storage.Put`, presign a 24 h GET.
  - Response 201: an array of `QrCode`, in the order of the request. Each has `format`, `content_url` and `generated_at`; for a bulk request **every item has the same `content_url`** (the one artifact), as documented in the contract change.
  - Audit `qr.generate` with the zone ids, format and layout in its details, **never the payloads**.
- **Rotation is not rebuilt here.** After `POST /admin/zones/{id}/rotate-qr` (10.1) the next `GET /qr-codes` shows the new `qr_token` and `generated_at`; the old printed code fails with `ZONE_QR_INVALID` in 10.2. The UI tells the admin to reprint (web spec §3.6).

**Tests**
- **Unit:** layout geometry (a grid layout of N zones makes `ceil(N / per_page)` pages); label truncation; the file-name slug; the unicode translator keeps a non-ASCII label intact.
- **HTTP** (per storage driver, `local` and `minio`, as 04.1)
  - One zone, `png`: the response has one item; fetching `content_url` returns `image/png`; decoding the PNG (a QR decoder as a **test-only** dependency, for example `github.com/makiuchi-d/gozxing`) gives exactly `qr_token` (AC-W35-9), and that payload verifies with 10.2's verifier.
  - `svg` is valid XML. Several zones with `png` → a ZIP with that many entries.
  - `pdf` with 17 zones in `grid-2x4` has 3 pages (count `/Type /Page` objects: page dictionaries are not compressed); `single` has one page per zone.
  - Selection and errors: none or two of the three selectors → `VALIDATION_ERROR`; a security zone with `zone_id` → `FEATURE_NOT_ENABLED`; a mix of valid and foreign ids → only the valid ones are produced; all foreign → `QR_NO_ZONES`; 201 zones → `QR_TOO_MANY_ZONES`; an unknown `format` or `layout` → `VALIDATION_ERROR`; inactive zones excluded unless `include_inactive`.
  - Authorization: a field user with `ememo.use` gets 403 `NOT_ADMIN`; a `PROJECT_ADMIN` sees and prints only their projects' zones; an `ORG_ADMIN` sees all cleaning projects.
  - Rotation: after `rotate-qr`, `GET /qr-codes` shows a different `qr_token`; the old payload is refused by the 10.2 verifier.
  - Idempotency: a replay returns the same `content_url` without writing a second object; the same key with another body → 409.
  - The field `GET /zones` response never contains the secret; the audit entry of `qr.generate` contains no payload.
  - Cross-tenant: org B's admin cannot list or print org A's zones (`QR_NO_ZONES` for ids, `ZONE_NOT_FOUND` for `zone_id`).

---

## Open questions

These need an answer from the product owner. None blocks 16.1 or 16.2 except where noted.

| # | Question | Recommended default |
|---|---|---|
| Q1 | Does anyone need a **personnel identity QR** (the old plan's `PERSIST-PERSON` card)? If yes: who scans it, and what happens? | **No**, built nothing (D16-7). If yes, it becomes its own phase with a spec: a signed payload (`QR_IDENTITY_SECRET`, HMAC-SHA-256, a key version in the payload so the secret can rotate), a named verifier and consumer, and a permission. When confirmed no, delete the `QR_IDENTITY_SECRET` mention from `.env.example`. |
| Q2 | MOB-15 lists "Generate QR" on mobile. Should a **DANRU** print codes from the phone? | **Admin only** for now (D16-8). If yes, add a `qr.generate` key to the 03.1 catalog (DANRU, scope `PROJECT`) and let the two routes accept it as an alternative to `RequireAdmin`. |
| Q3 | Is **revoke without regenerate** needed (for example a sticker is lost and the zone should stop accepting scans until reprinted)? | **No** (D16-9). If yes, it needs `patrol_zones.qr_revoked_at`, a check in 10.2's verifier, `rotate-qr` clearing it, and `QrCode.revoked` becomes meaningful. |
| Q4 | Should the **sender** see who read the memo (per recipient)? The web list shows "status baca per penerima", but `EMemo` only has `is_read` for the caller. | The data exists (`e_memo_recipients.read_at`). Not exposed; add a `read_receipts` field to the contract if wanted. |
| Q5 | **Admin tiers.** The web spec has five admin roles (Org Super Admin, Admin, Admin Cabang, Admin Klien, Admin Proyek); the plan has two (`ORG_ADMIN`, `PROJECT_ADMIN`, 01.3). "Admin Klien" is described as mostly read-only. | Map the first two to `ORG_ADMIN` and the others to `PROJECT_ADMIN` with `project_ids`. A read-only client admin is **not** modeled; it needs a role in 01.3 first. |
| Q6 | Should the sender be able to **edit or delete** after recipients have replied? Should an admin be able to delete someone else's memo (moderation)? | Sender only, any time, audited (16.2). Admin moderation is not built. |

---

## Out of scope (explicitly)

- Chemicals, MSDS and expiry (FEFO), the Master Cleaning Schedule, the Command Center and the area logbook. LOCK/03 names them, but gap design `07-adjustments-clarifications.md` lists them only as hypotheses (DL-71 is "proposed"). They need a decision and a spec first, and they will be a module of their own, not additions to this one.
- Attachments on replies, edit history of a memo, push delivery details (05.2 delivers whatever `Notify` queues), per-organization on/off switch for the cleaning features (D16-5).

---

## Follow-ups in other files

Found while writing this plan. They are small, and they are **not** done by this document:
- `03-access-control.md`: add `FEATURE_NOT_ENABLED` to the error-reasons line; export `access.CheckBusinessLine`.
- `04-media.md` (04.1): add `Storage.Put`.
- `10-patrol.md` (10.1): put the payload and renderer in `pkg/zoneqr`; add `qr_rotated_at`; note that `GET /zones` must never return the secret (already stated).
- `05-notifications.md`: list `EMEMO_CREATED` and `EMEMO_REPLIED` in the type tables (this module adds them).
- `docs/backend-modules.md`: line 85 and the graph (lines 61 to 62) list module 16's needs as "10, 14"; per this file it is 03, 04, 05 for 16.1/16.2 and 04, 10 for 16.3.
- `persist-ecosystem/docs/04-backend-core/01-data-model.md` §11: add `business_line` to `e_memos`, `deleted_at`, `local_id`, `synced_at`, the two recipient arrays, and the `e_memo_recipients` table.
