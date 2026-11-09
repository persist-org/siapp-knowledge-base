 # 05: Notifications, realtime and Personel events

**Purpose:** Three delivery channels: an in-app inbox, push notifications (FCM), and a WebSocket connection (essential for panic sirens). Also the receiver for change events from Persist Personel.
**Manual refs:** §2.5, Tabel 7 (List Notifikasi).
**Needs:** 03.
**Contract operations:** `GET /notifications` (list + delta sync), `GET /notifications/count`, `GET /notifications/{id}`, `POST /notifications/{id}/read`, `POST /notifications/read-all`. The WebSocket is outside the OpenAPI contract and follows `persist-ecosystem/docs/04-backend-core/03-realtime-websocket.md`.
**Error reasons:** `NOTIFICATION_NOT_FOUND`, `WEBHOOK_SIGNATURE_INVALID`.

**Types**

The 12 types from Tabel 7:

| Type | Recipient | Source |
|---|---|---|
| `LEAVE_REQUESTED` | DANRU | SiAPP (07.5) |
| `ABSENCE_MARKED` | Person | SiAPP (07.6) |
| `REPORT_SUBMITTED` | Team, or the DANRU for DANRU-only reports | SiAPP (11) |
| `INSTRUCTION_RECEIVED` | Instruction recipients | SiAPP (12) |
| `TEAM_CHANGED` (Perpindahan Regu) | Person | SiAPP (02.2 move) |
| `POSITION_CHANGED` (Pergantian Posisi) | Person | ⚠ **Personel event** (05.3): positions belong to Personel now. Also sent for a team role change (02.2). |
| `PATROL_SUBMITTED` | DANRU | SiAPP (10) |
| `PATROL_APPROVED` | Person | SiAPP (10) |
| `GUARD_SUBMITTED` | DANRU | SiAPP (10) |
| `GUARD_APPROVED` | Person | SiAPP (10) |
| `PANIC_TRIGGERED` | Team (+ project monitor) | SiAPP (09) |
| `OFFLINE_SUBMISSION_SENT` | Author | SiAPP (10, 11, 15) |

The manual's flows also imply these, which Tabel 7 does not list:

| Type | Recipient | Source |
|---|---|---|
| `LEAVE_DECIDED` | Requester | SiAPP (07.5), when the DANRU or an admin decides |
| `LEAVE_CANCELLED` | Whoever was told of the request (DANRU, or the admins) | SiAPP (07.5) |
| `INSTRUCTION_FEEDBACK` | Author | §3.2.4.5 |
| `SWAP_ASSIGNED` | Both people | Implied by §2.6.9 |
| `PANIC_RESOLVED` | Team | DL-31 |

**Rule for all modules:** call `notifications.Service.Notify()` **after** the commit. It never returns an error to the caller.

---

## Phase 05.1: Inbox

**Goal:** A notifications table, a single `Notify()` entry point, and the inbox endpoints.

**Build**
- **Table `notifications`** (tenant table, RLS)
  - Columns:
    - `id`, `organization_id`, `recipient_personnel_id`;
    - `type`: ENUM of the types above;
    - `title`, `body`;
    - `entity_type` and `entity_id` (both nullable);
    - `data` (jsonb);
    - `priority`: ENUM(`NORMAL`,`HIGH`);
    - `read_at` (nullable), `created_at`, `updated_at`, `deleted_at` (nullable; tombstones for delta sync).
  - Index: (`organization_id`, `recipient_personnel_id`, `read_at`, `created_at`) and (`organization_id`, `recipient_personnel_id`, `updated_at`) for delta sync.
- **`pkg/notifications/templates.go`**
  - `Render(t Type, v Vars) (title, body string)`: a `switch` over every type, with Indonesian text.
  - Labels follow the project's `business_line` (`DANRU` vs `Supervisor`, LOCK/03 §2): `Vars` carries the business line and `Render` uses the alias table from `persist-ecosystem/docs/00-overview/01-product-overview.md` §3.1.
  - The `exhaustive` linter enforces that every type has a case.
- **`Service.Notify(ctx, Input{ Type, RecipientIDs, Vars, Entity, Data, Priority, ExcludePersonnelID })`**
  - Exported and documented. The organization comes from the context.
  - Steps:
    1. Dedupe the recipients and remove `ExcludePersonnelID`.
    2. Insert with sqlc `:copyfrom` inside `WithTenant`.
    3. Call `Dispatcher.Dispatch(ctx, orgID, ids)`. It is an interface with a no-op default; Phases 05.2 and 05.3 fill it in.
  - ⚠ Log errors and never return them. Comment why: a notification failure must never fail the business action that caused it.
- **Endpoints**
  - `GET /notifications?unread_only=&page=&limit=`; with `since=`/`cursor=` it returns the delta shape (`deleted[]` + `DeltaMeta`, DL-57).
  - `GET /notifications/count`, `GET /notifications/{id}`, `POST /notifications/{id}/read`, `POST /notifications/read-all`.
  - A person sees and marks only their own notifications.

**Tests**
- Unit: every type renders a non-empty title and body in both business lines.
- HTTP:
  - `Notify` creates the rows.
  - List, count, delta sync and mark-read work.
  - Person A cannot mark person B's notification; cross-tenant ids return 404.

---

## Phase 05.2: Push delivery

**Goal:** Deliver notifications to devices through FCM, from a retrying background queue.

**Build**
- **Dispatcher.** The asynq dispatcher enqueues one task `push:deliver` per `Notify` call. The payload carries `organization_id` and the notification ids; the handler runs inside `WithTenant`.
- **Handler in `cmd/worker`.** The worker handler:
  - loads the notifications and the recipients' `devices`;
  - sends FCM multicast in chunks of 500;
  - deletes devices whose tokens FCM reports as unregistered or invalid;
  - lets asynq retry: `MaxRetry(3)` with its default backoff.
- **`push.Provider` interface.** Two implementations:
  - `FCMProvider`, configured from `FCM_SERVICE_ACCOUNT_BASE64` (iOS goes through APNs via FCM).
  - `LogProvider`, used when credentials are absent and in tests.
- **High priority.** ⚠ For `priority = HIGH` (panic):
  - put the task on the `critical` asynq queue, which gets a higher queue weight;
  - send with Android `priority: high`, `channelId: "panic"` and `sound: "siren"`, and iOS `interruption-level: critical` where allowed.
  - Comment that the channel and the sound must also exist in the mobile app, and that panic cannot be muted in notification settings (mobile spec).
- **Push data.** Include `notification_id`, `type`, `entity_type` and `entity_id` so the app can deep-link. No NIK or names beyond the rendered text.

**Tests**
- Unit: the handler deletes invalid tokens, using a fake provider.
- HTTP: `Notify` → run the handler inline with the task payload → the fake provider receives the expected tokens.

---

## Phase 05.3: Realtime WebSocket and Personel events

**Goal:** Authenticated WebSocket connections with rooms, across all API instances; and the receiver that turns Personel change events into cache invalidation and notifications.

**Build**
- **Endpoint `GET /api/v1/ws`**, subprotocol `persist.v1`, `coder/websocket`.
  - Auth: `Authorization: Bearer` (mobile), or a single-use ticket for browsers: `POST /ws/ticket` (authenticated) returns `{ ticket, expires_in: 30 }`, stored in Redis; the browser connects with `?ticket=`. ⚠ Comment why not `?token=` as the persist realtime doc allows: access tokens in URLs end up in proxy logs; a 30-second single-use ticket is harmless there. (Contract change: add `POST /ws/ticket`; propose the ticket to the realtime doc.)
  - On accept, send `hello` with `server_time`.
- **Rooms and messages** (persist realtime doc §4–5): `team.{id}.panic`, `project.{id}.panic`, `team.{id}.tracking`, `project.{id}.tracking`, plus a personal room for `notification.created`.
  - The client sends `subscribe` with room names; the server answers `subscribe.ack` with joined and denied rooms.
  - ⚠ Authorization per room: team panic rooms for current members; team tracking for that team's DANRU (`team.positions.view`); project rooms for admins whose scope covers the project. Internally every room name is namespaced with the organization id, so two tenants' `team.7.panic` never meet. Comment both rules.
  - Envelope `{ type, room, ts, msg_id, data }`. No NIK or full names in frames; use ids.
- **`realtime.Hub` (in-process).**
  - Tracks each local connection with its organization, person and joined rooms.
- Each connection has a buffered send channel (size 32). A slow client whose buffer is full gets disconnected, not blocked on. ⚠ Comment why: one stuck phone must not stall a panic broadcast.
- **Cross-instance fan-out.** ⚠ `realtime.Publisher.Publish(orgID, room, type, data)` publishes one JSON envelope to the Redis channel `rt:{orgID}`. Each instance subscribes with a pattern (`rt:*`) and delivers to matching local connections. Comment the trade-off: every instance sees every event; simple and fine at this scale.
- **Keepalive.** Ping every 20 s; close with `4408` if the pong is missing for 60 s. Close with `4401` on token expiry; the client refreshes and reconnects. Close codes per the realtime doc §3.3.
- **Dispatcher.** In addition to push, it publishes `notification.created` to each recipient's personal room.
- **Wire Phase 02.2:** `move` → `TEAM_CHANGED`; `change-role` → `POSITION_CHANGED`. `organization` depends on a small `notifier` interface, so it does not import `notifications`. Clients resubscribe to rooms after `TEAM_CHANGED`.
- **Personel webhook receiver** 🔌 (S6) `POST /api/v1/integrations/personel/events` — public route, authenticated by signature:
  - ⚠ Verify `X-Personel-Signature` (HMAC-SHA256 of the raw body with `PERSONEL_WEBHOOK_SECRET`, constant-time compare) and reject events older than 5 minutes (`timestamp` in the body). Store each `event_id` for 7 days and ignore repeats. Comment why: Personel delivers at least once, and a replayed event must not notify twice.
  - The organization comes from the event and is checked against `organizations`; the handler runs in `WithTenant`.
  - Events:
    - `employee.updated` → refresh the `personnels` row and `personel.Cached.Invalidate`.
    - `employee.deactivated` → `status = INACTIVE`, end all sessions (01.2), end active team memberships today.
    - `placement.changed` → invalidate the placement cache; if the person is no longer placed at a project where they have an active membership, notify the project admins (do not auto-remove: the admin decides).
    - `position.changed` → `POSITION_CHANGED` to the person.
  - Add the route to `auth.PublicRoutes` and mark it `security: []` with a `webhookSignature` security note in the contract.

**Tests**
- HTTP, with a websocket client in the test:
  - no token or a bad token is rejected (`4401`); a used or expired ticket is rejected;
  - a member receives `notification.created`;
  - a member cannot join another team's tracking room; a DANRU can;
  - ⚠ two organizations with the same team id: an event in org A never reaches org B's socket;
  - with two hubs sharing Redis, an event published on one is delivered on the other;
  - an admin move creates a `TEAM_CHANGED` inbox row.
- Webhook: a bad signature is rejected; a replayed `event_id` is ignored; `employee.deactivated` ends sessions and memberships; `position.changed` creates an inbox row.
