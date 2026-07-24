# API Specification
## ReDA Lab Resource Manager

**Document Status:** Draft v1.0
**Last Updated:** 2026-06-30

> REST-ish HTTP/JSON API exposed by the Core Backend. OpenAPI 3.0 compatible. Base URL: `https://<backend>/v1`. All bodies are `application/json` unless noted. Times are ISO-8601 UTC strings. Identifiers are UUIDv7 strings unless noted.

Companion documents: `SRS.md`, `SDD.md`, `database-erd.md`.

---

## 1. Conventions

### 1.1 Authentication & Authorization
| Header | Used by | Notes |
|---|---|---|
| `Authorization: Bearer <BOT_TOKEN>` | Bot Service | Fixed secret shared between Bot and Backend. |
| `Authorization: Bearer <ADMIN_TOKEN>` | Admin Web / External tools | Long-lived admin token issued via superadmin. |
| `Authorization: Bearer <MASTER_TOKEN>` | Master Node Controller | Per-master-node secret. |
| `X-Telegram-User-Id: <bigint>` | All Bot-era requests | The Telegram User ID of the human whose action is being performed. Backend resolves it to a `User`. |
| `X-Idempotency-Key: <uuid>` | State-changing requests | Allows safe retries (Telegram update re-delivery). |

RBAC enforcement is per-route (see "Auth" field on each endpoint).

### 1.2 Status Codes
- `200` success (with body)
- `201` created (with body)
- `202` accepted (async — e.g. booking approval triggers background jobs)
- `204` no content
- `400` validation error
- `401` unauthenticated
- `403` forbidden (role mismatch)
- `404` not found
- `409` conflict (e.g. duplicate active booking, illegal state transition)
- `422` semantic error (e.g. slot in the past)
- `429` rate limited
- `5xx` server error

### 1.3 Standard Error Body
```json
{
  "error": {
    "code": "BOOKING_CONFLICT",
    "message": "You already have an active booking.",
    "details": { "existing_booking_id": "0191ab..." }
  },
  "trace_id": "0191ab..."
}
```

### 1.4 Pagination
`GET` list endpoints accept `?limit=20&cursor=<opaque>`. Response:
```json
{ "items": [...], "next_cursor": "..." }
```
`next_cursor` is null when there is no more data.

### 1.5 Common Enums
- `category`: `HIGH_SPEC`, `LOW_SPEC`
- `pc_status`: `AVAILABLE`, `IN_USE`, `MAINTENANCE`, `OFFLINE`
- `booking_status`: `PENDING`, `APPROVED`, `POWER_ON_QUEUED`, `ACTIVE`, `REJECTED`, `CANCELLED`, `COMPLETED`, `COMPLETED_OVERRUN`, `NO_SHOW`, `NEEDS_MANUAL`, `RECLAIMED`, `WAITLISTED`
- `command_type`: `POWER_ON`, `GRACEFUL_SHUTDOWN`, `FORCE_SHUTDOWN`, `INSTALL_KEY`, `REMOVE_KEY`, `PROBE`
- `role`: `STUDENT`, `ADMIN`, `SUPERADMIN`

---

## 2. Identity & Registration

### 2.1 `POST /v1/users/register`
Register a new student account from a Telegram-initiated conversation. (Auth: Bot token.)

**Request**
```json
{
  "telegram_user_id": 123456789,
  "full_name": "Budi Santoso",
  "student_number": "2021xxx",
  "course_lab_group": "ML Research Group A",
  "preferred_contact": "@budi",
  "ssh_public_key": "ssh-ed25519 AAAA... budi@laptop",
  "locale": "en",
  "timezone": "Asia/Jakarta"
}
```
**Response `201`**
```json
{ "id": "0191...", "telegram_user_id": 123456789, "status": "PENDING", "role": "STUDENT" }
```
Errors: `409` telegram_user_id already registered; `422` invalid SSH pubkey.

### 2.2 `GET /v1/users/me`
(Headers: `X-Telegram-User-Id`.) Returns the calling user's profile.
```json
{ "id": "...", "telegram_user_id": 123456789, "role": "STUDENT", "status": "ACTIVE",
  "full_name": "...", "ssh_public_key_fingerprint": "SHA256:..." }
```

### 2.3 `PATCH /v1/users/me`
Update own profile (allowed fields: `full_name`, `course_lab_group`, `preferred_contact`, `ssh_public_key`, `locale`, `timezone`).

### 2.4 `GET /v1/users` — superadmin only
`?role=STUDENT&status=BLOCKED&limit=20&cursor=...` list users.

### 2.5 `PATCH /v1/users/{id}/role` — superadmin only
Body: `{ "role": "ADMIN" }`. Writes to `audit_log`.

### 2.6 `POST /v1/users/{id}/block` and `POST /v1/users/{id}/unblock` — superadmin only
Body: `{ "reason": "..." }`. Sets `users.status`.

---

## 3. Bookings

### 3.1 `POST /v1/bookings`
(Headers: `X-Telegram-User-Id`.) Submit a new booking. Auth: STUDENT (or higher).

**Request**
```json
{
  "category": "HIGH_SPEC",
  "workload_description": "Fine-tune a 7B model on a custom corpus (~6h).",
  "slot_duration_hours": 8,
  "slot_start": "2026-07-04T09:00:00Z",
  "allow_low_spec_substitute": false
}
```
Semantics:
- `slot_start` may be omitted/null to mean "ASAP".
- If `slot_start` would exceed `advance_booking_window_days` → `422`.
- If student already has an active or pending booking → `409 BOOKING_CONFLICT`.
- No-show penalty active → `422 NO_SHOW_PENALTY`.

**Response `201`**
```json
{
  "id": "0191...",
  "status": "PENDING",
  "student_id": "...",
  "category": "HIGH_SPEC",
  "slot_start": "2026-07-04T09:00:00Z",
  "slot_end":   "2026-07-04T17:00:00Z",
  "booked_at": "2026-06-30T08:00:00Z"
}
```

### 3.2 `GET /v1/bookings`
List bookings; filter by `status`, `category`, `student_id`, date range (`from`, `to`).
Auth:
- STUDENT: implicitly restricted to own bookings (server ignores `student_id`).
- ADMIN/SUPERADMIN: any.
Query: `?status=PENDING&limit=20&cursor=...`.

**Item** (reused in `GET /v1/bookings/{id}`)
```json
{
  "id": "0191...",
  "student": { "id": "...", "full_name": "Budi S." },
  "category": "HIGH_SPEC",
  "workload_description": "Fine-tune a 7B ...",
  "slot_start": "...", "slot_end": "...",
  "status": "APPROVED",
  "decided_by_admin_id": "...",
  "allocation": {
     "pc_id": "...", "pc_label": "PC-H1",
     "ssh_username": "labuser",
     "lan_ip": "10.0.0.11",
     "tunnel_address": "100.64.0.11"
  },
  "booked_at": "...", "decided_at": "...", "activated_at": "...", "ended_at": null
}
```

### 3.3 `GET /v1/bookings/{id}`
Auth: owning STUDENT or any ADMIN.

### 3.4 `POST /v1/bookings/{id}/approve`
Auth: ADMIN+. Body options: `{ "dry_run": false }`. Side effect:
1. Booking → `APPROVED`.
2. Run allocation; if a PC is found → create `allocation`, enqueue `POWER_ON` + `INSTALL_KEY` commands → booking → `POWER_ON_QUEUED` until master confirms → `ACTIVE`.
3. If no PC available → `WAITLISTED` (or `NEEDS_MANUAL` if no path).
4. Audit logged.

Response `202`:
```json
{ "id": "0191...", "status": "POWER_ON_QUEUED",
  "allocation": { "pc_label": "PC-H1", "mac": "aa:bb:cc:dd:ee:01" } }
```
Errors: `409` already decided; `422` no available PC.

### 3.5 `POST /v1/bookings/{id}/reject`
Auth: ADMIN+. Body: `{ "reason": "Workload not eligible for high-spec." }`. Booking → `REJECTED`, student notified.

### 3.6 `POST /v1/bookings/{id}/cancel`
Auth: owning STUDENT or any ADMIN. Booking → `CANCELLED`. If active: enqueue `GRACEFUL_SHUTDOWN` + `REMOVE_KEY`.

### 3.7 `POST /v1/bookings/{id}/extend`
Auth: ADMIN+. Body: `{ "extension_minutes": 60 }`. Validates against `policies.max_extension_minutes`.
Updates `slot_end`. Notifies student.

### 3.8 `POST /v1/admin/bookings/{id}/reclaim`
Auth: ADMIN+. Body: `{ "reason": "Public reason shown to student." }`. Booking → `RECLAIMED`, force shutdown path.

### 3.9 `GET /v1/bookings/waitlist`
Auth: ADMIN+. Returns waitlisted bookings for an admin to triage; oldest first.

---

## 4. PCs

### 4.1 `GET /v1/pcs`
List PCs. Auth: ADMIN+. Query: `?category=&status=`.

### 4.2 `GET /v1/pcs/{id_or_label}`
Returns inventory + live status (rolled up from the most recent `pc_status_log`).

### 4.3 `POST /v1/pcs` — superadmin only
```json
{ "label": "PC-L5", "hostname": "pcl5.lab.local",
  "mac_address": "aa:bb:cc:dd:ee:15", "lan_ip": "10.0.0.25",
  "category": "LOW_SPEC", "specs_summary": "i5, 16GB",
  "managed_by_master_id": "0191...", "ssh_user": "labadmin" }
```

### 4.4 `PATCH /v1/pcs/{id}` — admin+
Editable: `hostname`, `mac_address`, `lan_ip`, `category`, `specs_summary`, `ssh_user`, `status`.

### 4.5 `POST /v1/pcs/{id}/maintenance` / `POST /v1/pcs/{id}/release`
Auth: ADMIN+. Toggle `MAINTENANCE` status. A PC entering `MAINTENANCE` does not power off an existing active booking (admin must reclaim) but stops serving new allocations.

### 4.6 `POST /v1/pcs/{id}/power`
Auth: ADMIN+. Body: `{ "action": "on" | "off" | "restart" }` → enqueue commands accordingly.

### 4.7 `GET /v1/pcs/{id}/status-history`
Query: `?from=&to=&limit=...`. Returns `pc_status_log` rows.

---

## 5. Admin

### 5.1 `GET /v1/admin/audit`
Query: `?actor_id=&target_type=&target_id=&action=&from=&to=&limit=`. Returns `audit_log` rows (oldest/newest controlled by `?order=asc|desc`).

### 5.2 `POST /v1/admin/announce`
Auth: ADMIN+. Body: `{ "body": "Lab will be offline Sat 09:00–12:00." }`. Enqueues broadcast notifications to all active students; logs an `announcements` row and audit entry.

### 5.3 `GET /v1/admin/reports/usage`
Query: `?from=&to=`. Aggregates:
```json
{
  "from": "...", "to": "...",
  "total_bookings": 132,
  "approval_rate": 0.81,
  "no_show_rate": 0.07,
  "pc_utilization_hours": [
    { "pc_label": "PC-H1", "hours": 187 }, ...
  ],
  "top_students_by_bookings": [
    { "student_id": "...", "full_name": "...", "count": 9 }
  ]
}
```

---

## 6. Master Node Controller API
All require `Authorization: Bearer <MASTER_TOKEN>`. The path prefix is `/v1/master`.

### 6.1 `POST /v1/master/heartbeat`
Body: `{ "state": { "pc_states": [ { "pc_id": "...", "state": "ON", "has_active_ssh_session": true } ] } }`.
Updates `master_nodes.last_heartbeat_at`, upserts `pc_status_log` rows. Response `204`. The backend uses this also to surface live status in `/v1/pcs/{id}`.

### 6.2 `GET /v1/master/commands`
Long-poll for pending commands. Query: `?since=<command_id>&timeout=30`.
Returns at most N commands ordered by id. If none pending, holds the connection up to `timeout` seconds and returns `[]` when timeout reached.

**Response `200`**
```json
{ "commands": [
   {
     "id": 1042,
     "command_type": "POWER_ON",
     "payload": { "mac": "aa:bb:cc:dd:ee:01", "ip": "10.0.0.11", "ssh_user": "labadmin" },
     "target_pc_id": "0191..."
   }, ...
] }
```

### 6.3 `POST /v1/master/commands/{id}/dispatched`
Marks the command as `DISPATCHED` and stamps `dispatched_at`. Idempotent.

### 6.4 `POST /v1/master/commands/{id}/result`
Body:
```json
{ "status": "DONE" | "FAILED",
  "result": { "ip_reachable": true, "key_installed": true },
  "error_message": null }
```
Semantics by command type:
- `POWER_ON` DONE → booking transitions to `ACTIVE`; sends student notification (FR-20).
- `POWER_ON` FAILED → backend retries per `power_on_retries`, then `NEEDS_MANUAL` (UC-5).
- `INSTALL_KEY` DONE → sets `bookings.key_installed = true`.
- `GRACEFUL_SHUTDOWN`/`FORCE_SHUTDOWN` DONE → booking → `COMPLETED` or `COMPLETED_OVERRUN`; PC → `AVAILABLE`; notifications queued.
- `PROBE` → updates `pc_status_log`.

### 6.5 `POST /v1/master/pcs/{id}/status`
Already covered by heartbeat, but provided for on-demand pushes. Body: `{ "state": "ON", "has_active_ssh_session": false }`.

### 6.6 `POST /v1/master/reconnect`
Optional; master can ack that it has resumed after being offline. Backend flushes QUEUED commands older than a TTL.

---

## 7. Policies (runtime config)

### 7.1 `GET /v1/policies` — admin only
Returns the `policies` table.

### 7.2 `PATCH /v1/policies/{key}` — superadmin only
Body: `{ "value": [2, 4, 8] }`. Validates value schema per key (server-side). Writes audit entry.

---

## 8. Health & Misc

### 8.1 `GET /healthz`
Returns `200 { "ok": true, "db": "up", "master_nodes_online": 1 }` or `503` if DB down.

### 8.2 `GET /metrics`
Prometheus exposition format. Examples:
```
bookings_total{status="active"} 7
power_on_duration_seconds_bucket{le="60"} 4
master_node_up 1
command_queue_depth 2
```

### 8.3 `POST /v1/webhooks/telegram`
Receives Telegram webhook updates (only the Bot Service uses this). Validates the secret token set with the webhook (Telegram's `secret_token` mechanism). The Bot Service parses updates and forwards logical actions to the rest of `/v1/...` using its BOT_TOKEN.

### 8.4 `GET /docs` and `GET /openapi.json`
FastAPI auto-generated OpenAPI spec. Disabled in prod unless explicitly enabled.

---

## 9. Webhooks (Outbound, Telegram)
We push to Telegram from the notification outbox, not as inbound HTTP.
Per booking lifecycle the backend prepares a Telegram message via Bot Service:
- `BOOKING_PENDING_ADMIN` → all admins
- `BOOKING_APPROVED` → student
- `BOOKING_REJECTED` → student
- `PC_POWERED_ON` + SSH details → student
- `SLOT_END_WARNING` → student
- `SLOT_END_GRACE` → student
- `NO_SHOW_RELEASE` → student
- `FORCE_RECLAIM` → student
- `MASTER_OFFLINE_ALERT` → admins
- `WAITLIST_PROMOTION` → student
- `ANNOUNCEMENT` → all active students

Each is delivered at-least-once via the `notification_outbox` table + a worker.

---

## 10. Rate Limiting
- Bot Service: 30 req/s per source user.
- Master node: 10 req/s per master node (and enforced on the long-poll `timeout` cap of 60s).
- Admin Web: 5 req/s per admin.
- Write endpoints per-student: max 1 concurrent active-vs-pending booking; see SRS BR-3.

---

## 11. Versioning
- URL-prefixed: `/v1/...`. Breaking changes require `/v2/`.
- Non-breaking additions (new optional fields, new endpoints, new values in non-closed enums) are allowed without bumping.

---

## 12. Example: End-to-End HTTP Trace

```
# Bot submits a booking for user telegram_user_id=123456789
POST /v1/bookings
Headers:
  Authorization: Bearer <BOT_TOKEN>
  X-Telegram-User-Id: 123456789
  X-Idempotency-Key: 0192-...
Body: { "category": "HIGH_SPEC", "workload_description": "...",
        "slot_duration_hours": 4, "slot_start": null }
→ 201 { "id": "0191...", "status": "PENDING" }

# Admin (telegram_user_id=999) approves
POST /v1/bookings/0191.../approve
Headers:
  Authorization: Bearer <BOT_TOKEN>
  X-Telegram-User-Id: 999
→ 202 { "status": "POWER_ON_QUEUED", "allocation": { "pc_label": "PC-H1" } }

# Master node long-polls
GET /v1/master/commands?since=1040&timeout=30
Headers: Authorization: Bearer <MASTER_TOKEN>
→ 200 { "commands": [ { "id": 1042, "command_type": "POWER_ON",
    "payload": {...}, "target_pc_id": "0191..." } ] }

# Master reports success
POST /v1/master/commands/1042/result
Body: { "status": "DONE", "result": { "ip_reachable": true, "key_installed": true } }
→ 204

# Backend transitions booking ACTIVE, queues a student notification
# Bot picks up from outbox → sends Telegram message
```