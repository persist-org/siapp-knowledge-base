# 08: Location and presence

**Purpose:**
- Online/offline status and "Terakhir online" (last seen) for each person.
- GPS positions for the DANRU map (the whole team, a single member, or a whole project) and for the web tracking monitor.

**Manual refs:** §2.9.7, §2.9.8, §3.2.5.3, §3.2.5.4, Tabel 8 (Regu rows).

**Needs:** 03, 05.

**Contract operations:** `GET /teams/{id}/map`, `GET /personnel-tracking`. Live updates go over the WebSocket (`location.update` in, `location.broadcast` out; realtime doc §6). **Contract change:** `POST /location-logs` (batch upload for points recorded while offline) and `GET /personnels/{id}/position`.

**Error reasons:** `LOCATION_BATCH_TOO_LARGE`, `TRACKING_DISABLED`.

**Changes from v1.2 (modernization):**
- **Online status.** The manual counts a person as online when they are logged in. Here, a person is online if they were **seen within the last 5 minutes**. Any request, WebSocket message or location ping counts.
- **Logout** makes the person offline immediately.
- **Location history** is kept for 30 days by default (`persist-ecosystem/docs/08-meeting-notes/gap-designs/04-data-retention.md` proposes 90; change the constant when that DL is adopted).
- Tracking is accepted only while the project has `gps_tracking_enabled` (DL-48) and the person is on shift (privacy contract).

---

## Phase 08.1: Presence

**Goal:** Fast online/offline checks and last-seen times in the team member lists.

**Build**
- **`presence.Service`** (exported, doc comments):
  - `Touch(ctx, orgID, personnelID)`: sets the Redis key `org:{orgID}:presence:{personnelID}` to the current time, with TTL `onlineWindow = 5 * time.Minute`.
  - ⚠ Throttle the DB write: persist `personnels.last_seen_at` at most once per 60 s per person. Use `SET org:{orgID}:presence:persist:{id} 1 NX EX 60` as the gate. Comment why: it avoids one DB write per request, while keeping last-seen accurate to about a minute.
  - `MarkOffline(ctx, orgID, personnelID)`: deletes the Redis key.
  - `Many(ctx, orgID, ids) (map[int64]Status, error)`: reads with `MGET`, falling back to `personnels.last_seen_at`.
- **Middleware** after `auth.Require`: calls `Touch`.
  - Use a 50 ms context timeout. Log errors and ignore them; never fail the request.
  - ⚠ Comment why this is synchronous: a Redis SET costs under a millisecond, and running it inline avoids untracked goroutines.
- **Wiring:**
  - WebSocket messages (Phase 05.3) call `Touch`.
  - `logout` (Phase 01.2) calls `MarkOffline`.
- **Extend the Phase 02.4 responses:** members and team detail gain `online` and `last_seen_at`.

**Tests**
- Unit: the throttle gate, using Redis from the compose stack.
- HTTP:
  - After a request, the person shows as online in team detail; after logout, offline.
  - `last_seen_at` is persisted.
  - The same person id in two organizations has two independent presence keys.

---

## Phase 08.2: Location logs, live tracking and maps

**Goal:** Devices send GPS points, live positions reach the DANRU and the web monitor, and the maps can be queried.

**Build**
- **Table `location_logs`** (tenant table, RLS), partitioned by month on `logged_at` (realtime doc §6):
  - `id`, `organization_id`, `personnel_id`, `logged_at`, `latitude`, `longitude`, `accuracy_meters`, `battery_pct` (nullable), `created_at`.
  - UNIQUE (`organization_id`, `personnel_id`, `logged_at`) (must include the partition key).
  - Partitions: created 3 months ahead by the monthly task `location:partitions`.
- **Table `personnel_last_locations`** (tenant table, RLS): `organization_id`, `personnel_id` (PK together), `logged_at`, `latitude`, `longitude`, `accuracy_meters`.
- **`location.Ingest(ctx, personnelID, points)`** (shared by REST and WebSocket):
  - Reject when the person has no current shift at a project with `gps_tracking_enabled` → `TRACKING_DISABLED` (REST) or an `error` frame (WS). Comment the privacy contract.
  - Drop points older than 24 h or more than 5 min in the future.
  - Insert with `:copyfrom` into a temp table, then `INSERT … SELECT … ON CONFLICT DO NOTHING`, so duplicates are ignored.
  - ⚠ Last-location upsert: `INSERT … ON CONFLICT (organization_id, personnel_id) DO UPDATE SET … WHERE EXCLUDED.logged_at > personnel_last_locations.logged_at`. Comment why: offline batches can arrive out of order and must never move a person "back in time".
  - Then `presence.Touch`, and publish `location.broadcast` to `team.{id}.tracking` and `project.{id}.tracking` for the newest point, at most once per person per 2 s (debounce in Redis).
- **WebSocket `location.update`** frames call `Ingest` with one point.
- **`POST /location-logs`** `{ points: [{ logged_at, latitude, longitude, accuracy_meters, battery_pct? }] }`: at most 100 points, otherwise `LOCATION_BATCH_TOO_LARGE`; responds `{ accepted, rejected }`.
- **Map endpoints** (`team.positions.view`, target in scope). Each returns **online** members only, with last location, name, photo and role:
  - `GET /teams/{id}/map`
  - `GET /personnel-tracking?project_id=` (also used by admins with project scope, for the web monitor; supports `team_id`)
  - `GET /personnels/{id}/position` (contract change): the target must be in a team within scope (§3.2.5.4 "Lihat di Peta").
- **Periodic task `location:retention`** (daily): drop partitions entirely older than `locationRetention = 30 days`. ⚠ Comment why partitions are dropped instead of rows deleted: it is instant and leaves no bloat.

**Tests**
- HTTP:
  - An out-of-order batch keeps the newest last location; a duplicate point is ignored.
  - Points are refused when the project has tracking off.
  - A member gets 403 on the map endpoints; a DANRU sees online members only.
  - A DANRU's WebSocket in `team.{id}.tracking` receives `location.broadcast`; another organization's socket does not.
  - Retention drops an old partition.
