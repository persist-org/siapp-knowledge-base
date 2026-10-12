# Backend API Response Format

**Source of truth**: `persist-ecosystem/contracts/openapi.yaml` · full reference: `syap/docs/api-response.md`

## 🎯 One Rule
Every `/api/v1/*` response is a JSON **envelope** with `success` at the top. Branch on `success` first; never expect a bare object or array.

## 📦 The Four Shapes

| Shape | When | Body |
| :--- | :--- | :--- |
| **Success** | one resource or an action | `{ success: true, data, meta }` |
| **List** | paged collection | `{ success: true, data: [], pagination, meta }` |
| **Delta** | offline sync (`since`, `cursor`) | `{ success: true, data: [], deleted: [], meta: { server_time, next_cursor, has_more } }` |
| **Error** | anything failed | `{ success: false, error: { code, message, details?, errors? }, meta }` |

### Success
```json
{ "success": true, "data": { "id": 42, "name": "Regu A" }, "meta": { "request_id": "req_8f2c1a" } }
```
`200` (read/update/delete/action), `201` (create). Nothing to return → `"data": null`.

### List
```json
{
  "success": true,
  "data": [ { "id": 5501, "subject": "Laporan lapangan" } ],
  "pagination": { "page": 1, "limit": 20, "total": 143, "total_pages": 8 },
  "meta": { "request_id": "req_8f2c1a" }
}
```
`page` default 1, `limit` default 20 (max 100). Empty → `"data": []`.

### Delta
```json
{
  "success": true,
  "data": [ { "id": 9001, "updated_at": "2026-10-12T02:59:10Z" } ],
  "deleted": [ { "id": 8800, "deleted_at": "2026-10-12T02:00:00Z" } ],
  "meta": { "request_id": "req_8f2c1a", "server_time": "2026-10-12T03:00:00Z", "next_cursor": "eyJpZCI6OTAwMX0", "has_more": true }
}
```
Save `server_time` and send it as the next `since`. `has_more` → call again with `cursor`. No `pagination` here.

### Error
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "Akun Anda terhubung dengan perangkat lain. Hubungi HR untuk ganti perangkat.",
    "details": { "reason": "AUTH_DEVICE_BOUND" }
  },
  "meta": { "request_id": "req_8f2c1a" }
}
```
*   `code`: broad category from a fixed enum. Branch on it.
*   `message`: Indonesian, safe to show as is.
*   `details.reason`: the exact cause (`AUTH_*`, `ATTENDANCE_*`, …), for special screens.
*   `errors[]`: per-field problems, only with `VALIDATION_ERROR` (`{ field, code, message }`).

## 🚦 Error Codes

| Code | HTTP | Client action |
| :--- | :--- | :--- |
| `BAD_REQUEST` | 400 | Bug: body is not JSON |
| `UNAUTHENTICATED` | 401 | Back to login |
| `TOKEN_EXPIRED` | 401 | `POST /auth/refresh` once, retry |
| `FORBIDDEN`, `ROLE_DENIED`, `SCOPE_DENIED` | 403 | Show message |
| `NOT_FOUND` | 404 | Show message (also used for other organizations' data) |
| `CONFLICT`, `DUPLICATE` | 409 | Reload / show message |
| `PAYLOAD_TOO_LARGE` | 413 | Body over 1 MB |
| `VALIDATION_ERROR` | 422 | Highlight `errors[]` fields |
| `UNPROCESSABLE`, `UNDERSTAFFED` | 422 | Business rule said no; show message |
| `RATE_LIMITED` | 429 | Wait `Retry-After` seconds |
| `INTERNAL_ERROR` | 500 / 503 | Retry later; report `meta.request_id` (503 = Personalia down) |

## 🔐 Login Result
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs…",
    "refresh_token": "7.Xk2v…",
    "token_type": "Bearer",
    "expires_in": 900,
    "user": { "id": 15, "organization_id": 7, "nik": "************5678", "first_name": "Budi" }
  },
  "meta": { "request_id": "req_8f2c1a" }
}
```

| Login failure | HTTP | `details.reason` |
| :--- | :--- | :--- |
| Wrong email / password | 401 | `AUTH_INVALID_CREDENTIALS` |
| Employee inactive | 403 | `AUTH_ACCOUNT_DISABLED` |
| Bound to another phone | 403 | `AUTH_DEVICE_BOUND` |
| Web login without web role | 403 | `NOT_ADMIN` |
| Organization suspended | 403 | `AUTH_ORGANIZATION_SUSPENDED` |
| Personalia down | 503 | `PERSONEL_UNAVAILABLE` |

## 🧾 Conventions
*   **Keys**: `snake_case` · **IDs**: int64 · **Money**: `"125000.00"`
*   **Timestamps**: UTC with `Z` · **Dates**: `"YYYY-MM-DD"` in the project timezone
*   **NIK**: masked (`************1234`)
*   **`meta.request_id`**: on every response (and `X-Request-Id` header); quote it in bug reports
*   **Idempotent replay**: same `Idempotency-Key` + same body → the first response again, with `Idempotent-Replayed: true`
