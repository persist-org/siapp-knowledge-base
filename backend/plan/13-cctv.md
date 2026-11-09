# 13: CCTV

**Purpose:** A registry of cameras per project and team, per-role access settings, and short-lived, secure playback for the app. The 1-up and 4-up camera layouts are handled by the app only.
**Manual refs:** §2.11, §3.2.7, Gambar 11.
**Needs:** 03.
**Contract operations:** `/admin/cctvs` (CRUD), `PATCH /admin/cctvs/{id}/access`, `GET /admin/cctvs/search`, `GET /cctvs`. **Contract change:** `POST /cctvs/{id}/playback`.
**Error reasons:** `CCTV_NOT_FOUND`, `CCTV_GATEWAY_ERROR`, `CCTV_TOKEN_INVALID`.

**Open decision D7: which camera/NVR vendor and protocol?**
- Phase 13.1 does not depend on the vendor.
- Phase 13.2 assumes **MediaMTX** as the gateway (RTSP in; HLS and RTSP out). Confirm before building it.
- ⚠ Alignment note for the mobile team: the PERSIST mobile spec plays RTSP directly with libVLC. With the gateway, libVLC plays the gateway's URL instead (it supports both HLS and RTSP), so camera credentials never reach the device. This is the reason to keep the gateway.

---

## Phase 13.1: Camera registry and access

**Goal:** Admins register cameras with credentials encrypted at rest and choose who may view them. Field users list the cameras they are allowed to view.

**Build**
- **`pkg/platform/secretbox`**: AES-256-GCM `Seal` and `Open` using `crypto/aes` and `crypto/cipher` (DL-12).
  - The key comes from `CCTV_ENCRYPTION_KEY` (32 bytes, base64).
  - ⚠ Comment the stored format `v1:{nonce}:{ciphertext+tag}` (base64url parts), why every value gets a fresh random nonce, and that the organization id is bound as additional data so a ciphertext copied to another tenant's row fails to open.
- **Table `cctvs`** (tenant table, RLS):
  - `id`, `organization_id`, `project_id`, `team_id` (nullable), `name`, `location_note` (nullable)
  - `source_url_encrypted` (text)
  - `gateway_path` (UNIQUE, `org{orgId}-cam{id}`)
  - `sort_order`, `active`, timestamps, `deleted_at`
- **Table `cctv_access`** (tenant table, RLS): `organization_id`, `cctv_id`, `team_role` ENUM(`DANRU`,`MEMBER`), PK (`organization_id`, `cctv_id`, `team_role`). Empty means "everyone with `cctv.view`". ⚠ Comment how this narrows the permission (web spec WEB-31 per-role access).
- **Admin endpoints**: `/admin/cctvs` CRUD and `/admin/cctvs/search?q=&project_id=`:
  - Accept `source_url` as plain text and store it encrypted.
  - Responses only return a masked URL, e.g. `rtsp://***@10.0.0.5/stream1`. Credentials are never returned or logged.
  - `PATCH /admin/cctvs/{id}/access` `{ team_roles: [] }`.
- **`GET /cctvs?project_id=&team_id=`** (`cctv.view`, filtered by scope and `cctv_access`) returns `[{ id, name, team_name, location_note }]`.
  - A camera with no `team_id` is visible to anyone whose scope covers its project.

**Tests**
- **Unit:**
  - Seal/Open round-trip; tampered ciphertext fails; wrong key fails; another organization's additional data fails.
- **HTTP:**
  - Admin responses mask the URL.
  - The field list respects scope and per-role access.
  - Cross-tenant: 404.

---

## Phase 13.2: Playback via the stream gateway (confirm vendor first)

**Goal:** The app gets a stream URL that expires within minutes. Camera credentials never reach the device.

**Build**
- **`docker-compose.yml`**: add `bluenviron/mediamtx` with:
  - `authMethod: http`
  - `authHTTPAddress` pointing at the API's internal auth endpoint (below)
  - the control API reachable only on the internal network
- **`cctv.Gateway` interface**, implemented by `MediaMTXGateway` using `net/http` against MediaMTX's control API:
  - On camera create or update: `POST /v3/config/paths/replace/{gatewayPath}` with the decrypted source and `sourceOnDemand: true`.
  - On camera delete: remove the path.
  - On failure: return `CCTV_GATEWAY_ERROR`, and never log the source URL.
- **`POST /cctvs/{id}/playback`** (`cctv.view` in scope) returns `{ hls_url, rtsp_url, expires_at }`.
  - URLs point at `${CCTV_GATEWAY_PUBLIC_URL}/{gatewayPath}/…?token={t}`.
  - `t` is an HMAC-signed `{ org, cctv_id, personnel_id, exp: now+5m }`.
- **`POST /internal/cctv/auth`**: no user auth, but it requires the header `X-Internal-Secret` (`CCTV_GATEWAY_SHARED_SECRET`, checked with `hmac.Equal`). Not exposed through the public proxy.
  - MediaMTX calls it with `{ path, action, query }`.
  - It allows only `action = read` with a valid, unexpired token for that path. Everything else gets 401.
  - ⚠ Comment the full flow:
    1. The app asks the API for playback.
    2. The app requests the stream from the gateway.
    3. The gateway calls the API's auth callback.
    4. The stream plays.

**Tests**
- **Unit:** token sign/verify, expiry, path mismatch, organization mismatch.
- **HTTP:**
  - A playback URL is issued within scope; a member excluded by `cctv_access` gets 403.
  - The auth callback rejects a missing secret, a wrong path and an expired token.
  - A fake `Gateway` records the call on camera create.
