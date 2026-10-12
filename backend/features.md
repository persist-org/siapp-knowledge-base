# Backend Features

Status: 🔄 exists in the first build (`/siapp/internal/...`), being moved and adapted · 📝 planned. Code locations are where the rebuild puts it. Plans live in `syap/docs/plan/NN-*.md`.

| Domain | Feature | Code Location | Status | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Platform** | Tenant isolation | `pkg/platform/database` | 📝 | Postgres RLS per organization, single `WithTenant` data path. |
| **Platform** | Contract & envelopes | `pkg/platform/httpx`, `internal/apigen` | 📝 | Generated types from the PERSIST contract, standard success/list/error envelopes. |
| **Platform** | Idempotency & audit | `pkg/platform/idem`, `pkg/platform/audit` | 🔄 | Offline replay with `Idempotency-Key`; append-only audit log. First build: `internal/platform/{idem,audit}`. |
| **Platform** | Personalia client | `pkg/platform/personel` | 📝 | Integration-key client with circuit breaker and Redis cache, plus a local stub. |
| **Auth** | Login via Personalia | `pkg/identity` | 📝 | Email + password checked by Personalia, one bound phone, SiAPP's own tokens. Replaces the first build's local-password login. |
| **Auth** | Sessions | `pkg/platform/auth` | 🔄 | Refresh rotation, reuse detection, Redis revocation list, default-deny middleware. First build: `internal/platform/auth` (`sessions.go`, `tokens.go`, `middleware.go`, `revocations.go`). |
| **Auth** | Web roles & profile | `pkg/access`, `pkg/identity` | 📝 | `ORG_ADMIN` / `PROJECT_ADMIN`, `GET /auth/me`, push-token devices, daily people resync. |
| **Organization** | Projects, teams, shifts | `pkg/organization` | 📝 | Projects from Personalia with local security settings; teams (regu), DANRU/member roles, shift schedules. |
| **Access** | Permission matrix | `pkg/access` | 📝 | Satpam/DANRU rights as data per organization; self/team/project scopes. |
| **Media** | Uploads | `pkg/media` | 📝 | Presigned uploads with capture metadata as anti-fraud evidence. |
| **Notifications** | Inbox, push, realtime | `pkg/notifications` | 📝 | Delta-sync inbox, FCM (panic channel), WebSocket rooms, Personalia webhook receiver. |
| **Master Data** | Categories & contacts | `pkg/masterdata` | 📝 | Report/instruction categories, panic types, emergency contacts, letterheads. |
| **Attendance** | Check-in/out | `pkg/attendance` | 📝 | Selfie + rotating QR, geofence, late rules, outbox push to Personalia, leave with DANRU decision, shift swaps. |
| **Presence** | Live tracking | `pkg/presence` | 📝 | Online status, partitioned location logs, live map rooms. |
| **Panic** | Emergency alert | `pkg/panic` | 📝 | Idempotent trigger, team + project broadcast, follow-up report, resolve. |
| **Patrol** | Zones & scans | `pkg/patrol` | 📝 | Printable QR zones, geofenced scans, patrol interval, progress colours. |
| **Reports** | Field reports | `pkg/reports` | 📝 | Offline-capable, DANRU-only visibility, search and filters. |
| **Instructions** | Tasking | `pkg/instructions` | 📝 | Multi-project/team targeting, read receipts, feedback. |
| **CCTV** | Camera playback | `pkg/cctv` | 📝 | Encrypted camera registry, short-lived playback via MediaMTX. |
| **Analytics** | Summaries & exports | `pkg/analytics` | 📝 | 13 report types, Excel/PDF/CSV with letterhead, dashboard. |
| **Assets** | Inventory | `pkg/assets` | 📝 | Conditions, per-project inventory, field asset reports. |
| **Cleaning** | E-Memo, QR | `pkg/cleaning` | 📝 | Cleaning-only features behind the `business_line` gate. |

## API Structure
The API is versioned under `/api/v1` and follows the PERSIST contract exactly:
*   `/api/v1/health`, `/api/v1/auth/login`, `/api/v1/auth/refresh`: the only public endpoints.
*   `/api/v1/<resource>`: field app endpoints, checked by permission and scope.
*   `/api/v1/admin/**`: web dashboard endpoints, gated by a web role.
*   Every response uses the PERSIST envelopes: see [api-response.md](api-response.md).

## Security Features
*   **No credentials stored**: passwords stay in Personalia; refresh tokens are stored only as SHA-256 hashes.
*   **Default-deny auth**: every route needs a SiAPP token unless it is on an explicit allowlist, enforced by a test.
*   **Tenant isolation**: Postgres RLS on every tenant table; cross-tenant tests for every endpoint.
*   **One phone per person**: device binding recorded in Personalia, reset by HR.
*   **Rate Limiting**: login, refresh and panic.
*   **Input Validation**: generated contract types + `go-playground/validator`.
*   **Safe logs**: never tokens, passwords, NIK, camera URLs or raw locations.
