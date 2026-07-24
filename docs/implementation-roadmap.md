# Implementation Roadmap
## ReDA Lab Resource Manager

**Document Status:** Draft v1.0
**Last Updated:** 2026-06-30

> Sequenced build plan for the system described in `SRS.md`, `SDD.md`, `database-erd.md`, `api-specification.md`. Designed for a small team (1–2 backend devs + 1 part-time lab engineer). Total target v1 effort: ~6 weeks part-time / ~3 weeks full-time.

---

## 1. Guiding Principles
1. **Vertical slices first.** Each phase delivers a demonstrable end-to-end flow on real hardware, even if narrow.
2. **Safety nets early.** Power management must include the idle auto-off sweep from Phase 1 to avoid permanently-on lab PCs.
3. **Test on real lab hardware by Phase 2.** Tailscale/WoL combos should be validated against a real lab PC before any further features land.
4. **No premature optimization.** Single-process backend, one master node, PostgreSQL. Scale concerns deferred (see SDD §12).

---

## 2. Phase Map

| Phase | Goal | Duration | Deliverable |
|---|---|---|---|
| 0 | Foundations: repo, infra, schema | 3–4 days | Skeleton + migrations + health endpoint |
| 1 | Booking + manual approval (Telegram) | ~1 week | Student can submit; admin approves in Telegram; **no power automation yet** |
| 2 | Power automation: master node, WoL, ephemeral SSH keys | ~1 week | Approval → PC powers on → student SSHs in → auto shutdown at slot end |
| 3 | Hardening: waitlist, reallocation, no-show, audit/UI | ~1 week | Resilient to edge cases of SRS §6 |
| 4 | Polish & ops: dashboards, metrics, documentation, rollout | ~1 week | Production-ready behind feature flag |

Then optional Phase 5+: GPU quotas, multi-lab, SSO, persistent home directories.

---

## 3. Phase 0 — Foundations

### 3.1 Work
- [ ] Initialize repo: `backend/`, `bot/`, `master_node/`, `docs/`, `deploy/`.
- [ ] `docker-compose.yml` with `backend`, `db` (Postgres), `bot` services.
- [ ] FastAPI skeleton with `/healthz`, config loader (`pydantic-settings`), `structlog`.
- [ ] SQLAlchemy 2.0 async + Alembic. Initial migration `0001_init` creating **all** tables from `database-erd.md`.
- [ ] RBAC dependency (`require_role`), `X-Telegram-User-Id` resolver.
- [ ] CI: GitHub Actions running `ruff`, `mypy`, `pytest`.
- [ ] Provision cloud VM (smallest needed, 1 vCPU/1GB is fine).
- [ ] Provision Tailscale tailnet (or Cloudflare Tunnel account); on lab master PC, install Tailscale and confirm reachability to a stub backend endpoint.
- [ ] Create the Telegram bot via `@BotFather` (token, username, description). Store token in `.env` (and in CI secrets later).

### 3.2 Acceptance
- `docker compose up` brings backend + DB up; `curl /healthz` returns `{"ok": true}`.
- Alembic upgrade creates all tables; `pylideploy` sanity inserts a `policies` row.
- Tailscale ping from cloud VM → lab master PC succeeds; reverse ping succeeds (mesh).
- Bot replies `/start` with a help message.

---

## 4. Phase 1 — Booking + Manual Approval (Telegram only)

### 4.1 Work
- [ ] Implement `POST /v1/users/register`, `GET /v1/users/me`.
- [ ] Implement `POST /v1/bookings`, `GET /v1/bookings`, `GET /v1/bookings/{id}` with role scoping.
- [ ] Implement `POST /v1/bookings/{id}/approve` and `/reject` (still NO power-on).
- [ ] Notification outbox worker (Telegram push via Bot Service).
- [ ] Bot flows:
  - `/start`, `/help`
  - `/register` (conversational form)
  - `/book` (category, workload, duration, ASAP/future)
  - `/mybookings`, `/cancel`
  - Admin inline keyboard on pending booking (Approve / Reject + reason prompt)
  - `/admin pending`, `/admin status` (read-only)
- [ ] Policies table seeded; `slot_duration_options`, `advance_booking_window_days`, `max_active_bookings_per_student`.
- [ ] Basic audit log writing for booking state transitions.

### 4.2 Acceptance
- A student can register, submit a booking, see it pending, and cancel.
- An admin receives the Telegram notification and approves/rejects; the student is notified of the decision (with the reject reason).
- If a non-admin tries to approve via direct API call → 403.

### 4.3 Out of scope
- Any power-on/off behavior. Bookings approved in Phase 1 simply stay in "APPROVED — manual setup" with a bot message instructing the student to ask the lab to power on the PC. (We use Phase 1 as the production freeze-camp: real approval flow without committing to power management.)

---

## 5. Phase 2 — Power Automation (the core wedge)

### 5.1 Work
- [ ] Build the Master Node Controller (`master_node/`) as a Python systemd service:
  - `python-telegram-bot` NOT needed; uses `httpx` long-poll + `wakeonlan` + `paramiko`.
  - Long-polls `GET /v1/master/commands`.
  - Heartbeat sender thread.
  - Local safety clock: shut down any PC whose active slot end is past and no recent heartbeat.
- [ ] Master node endpoints:
  - `POST /v1/master/heartbeat`
  - `GET /v1/master/commands` (long-poll with timeout)
  - `POST /v1/master/commands/{id}/dispatched`
  - `POST /v1/master/commands/{id}/result`
- [ ] Backend scheduling:
  - On `approve` → allocate → enqueue `POWER_ON` + `INSTALL_KEY`; booking `APPROVED` → `POWER_ON_QUEUED`.
  - APScheduler job at slot end for `GRACEFUL_SHUTDOWN` + `REMOVE_KEY`.
  - APScheduler job at `slot_end - warning` to send `SLOT_END_WARNING`.
  - APScheduler idle auto-off sweep (every 30 min) → `FORCE_SHUTDOWN` any PC with no active booking for > idle threshold.
- [ ] Allocation service (`allocation_service.pick`) per SDD §4.3 query.
- [ ] WoL execution + reachability probe (SSH `who`/`echo` ping) with timeout and retry.
- [ ] Ephemeral SSH key: snapshot to `allocations.ssh_pubkey_snapshot`; master appends/removes the exact line on `INSTALL_KEY`/`REMOVE_KEY`.
- [ ] End-to-end smoke test against a **single real high-spec PC + master node** in the lab.

### 5.2 Acceptance (matches SRS §7 acceptance criteria)
1. Student `/book` → admin Approve → within 5 minutes the PC is SSH-reachable from the master node.
2. Student receives a Telegram message with PC label, IP, SSH username, slot end.
3. Student can SSH in using their registered key.
4. Telegram warns 10 min before slot end; at slot end the PC gracefully shuts down; key removed; booking `COMPLETED`; PC returns to `AVAILABLE`.
5. Audit log shows approval, allocation, power-on, power-off events.

### 5.3 Risk-callouts
- WoL over wireless NICs is unreliable; lab PCs must be Ethernet.
- Lab firewall/NAT: must be solved before Phase 2 starts (Tailscale verified in Phase 0).
- Master node needs passwordless sudo SSH to each lab PC for `shutdown` and `authorized_keys` writes.

---

## 6. Phase 3 — Resilience, Waitlist, Reallocation, No-show

### 6.1 Work
- [ ] Waitlist (`WAITLISTED`) for `HIGH_SPEC` requests when no PC available + auto-promote on availability (APScheduler job).
- [ ] Power-on failure → reallocation up to `power_on_retries`; on exhaustion → `NEEDS_MANUAL` + admin alert.
- [ ] Master node offline detection (heartbeat stale 3 min) + queue replay on reconnect; message to admins.
- [ ] No-show detection job (FR-26) with config `no_show_grace_minutes`; on hit → `NO_SHOW`, shutdown PC, `REMOVE_KEY`, record `no_show_penalties`.
- [ ] No-show penalty enforcement on `POST /v1/bookings` (BR-4).
- [ ] Force-reclaim admin flow with mandatory public reason.
- [ ] Slot extension admin flow.
- [ ] Slot-end grace (`slot_end_grace_minutes`) with active-SSH detection.
- [ ] Telegram outage resilience: outbox backoff (up to 24h), digest on reconnect.
- [ ] Idle auto-off hardening: log every triggered shutdown into audit.
- [ ] Unit tests for the allocation algorithm and state machine.

### 6.2 Acceptance
- Simulate a PC that won't power on → backend reallocates to next PC and recovers.
- Kill the master node → backend marks it offline within 3 min, admins alerted, existing bookings preserved; restart master → queued commands drained.
- Simulate a no-show (no SSH session for 20 min) → booking auto-released, PC off.
- A waitlisted high-spec booking auto-promotes within 60s of a high-spec PC being released by another student.

---

## 7. Phase 4 — Polish & Ops

### 7.1 Work
- [ ] Read-only admin web dashboard (`/admin` Jinja2 + HTMX, or static SPA): current bookings, PC grid, audit log. Auth via admin token (Tg-login widget optional).
- [ ] `GET /v1/admin/reports/usage` aggregation endpoint + a CSV export button.
- [ ] `/metrics` Prometheus endpoint; alert rules for: `master_node_up == 0`, `command_queue_depth > 50`, `power_on_failure_rate > 25%`.
- [ ] Telegram bot copy externalized to `app/locales/{en,id}.yaml`.
- [ ] Runbook (`docs/RUNBOOK.md`): common incidents (master node offline, all PCs unreachable, Telegram outage).
- [ ] Backup script + restore drill: `pg_dump` → S3-compatible storage; restore validated in dev.
- [ ] Security review checklist:
  - Bot ignores unregistered users (only a `/help`/`/register` reply).
  - Master token rotated; rotation procedure documented.
  - No ephemeral SSH key left in any PC across restart.
  - Audit log immutability trigger present.
- [ ] Load test: simulate 100 concurrent booking submissions and 50 inline-button callbacks; no 5xx.

### 7.2 Rollout Plan
1. Internal dogfooding: 1 lab admin + 5 friendly students for 1 week.
2. Fix top issues from dogfooding. Cut v1.0.
3. Public announcement to the cohort via `/admin/announce`.
4. Two-week hypercare: on-call rotation between the two engineers; incident template ready.

### 7.3 Acceptance
- All P0/S0 SRS requirements covered and tested.
- OpenAPI docs published (`/docs`).
- Runbook + architecture diagrams (export from SDD)committed.

---

## 8. Cross-Cutting Workstream Mix

| Stream | Owner | Spans Phases |
|---|---|---|
| Backend & DB | Backend dev | 0→4 |
| Bot UX (Telegram flows) | Backend dev | 1→4 |
| Master node agent + lab hardware | Lab engineer | 0→3 |
| Infra (cloud VM, Tailscale, secrets) | Lab engineer | 0→4 |
| Testing & CI | Both | 0→4 |
| Documentation | Both | 0→4 |
| Security review | Backend dev | 3→4 |
| Rollout & comms | Lab admin / superadmin | 4 |

---

## 9. Work Breakdown (Spec-level Features → Stories)
The following user stories live behind each phase. Refer to `SRS.md` §2.2 and §5 for the full story list.

**Phase 0** — Stories: setup + hello-world Telegram reply.
**Phase 1** — Stories: As a Student I register; As a Student I submit a booking; As a Student I cancel; As an Admin I approve/reject; As the System I notify both parties.
**Phase 2** — Stories: As the System I auto power-on after approval; As the System I auto power-off at slot end; As the System I install/remove ephemeral SSH keys; As a Student I receive SSH details.
**Phase 3** — Stories: As the System I reallocate on PC failure; As the System I waitlist when out of GPUs; As the System I handle no-shows; As an Admin I force-reclaim a PC; As an Admin I extend a slot; As the System I survive a master node outage.
**Phase 4** — Stories: As an Admin I view a dashboard; As a Superadmin I export usage reports; As ops I receive alerts; As ops I follow a runbook.

---

## 10. Definition of Done (per phase)
- All acceptance criteria pass when run on real lab hardware or in a hardware-mocked local environment (a Docker "fake master node" that simulates PCs).
- CI green: `ruff`, `mypy --strict`, `pytest` (≥80% coverage on new modules).
- Migration is forward-only and reversible (downgrade tested).
- New FR IDs added to CHANGELOG with links to commits.
- Docs updated (RERDER relevant section of SDD/ERD/API if behavior shifts).

---

## 11. Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Lab PCs unreliable for WoL (BIOS settings, dormancy) | Phase 2 blocked | Phase 0 lab-bench test of WoL on one PC; document required BIOS settings. |
| University blocks Tailscale/Cloudflare Tunnel | Master node can't reach backend | Fall back to a cpsrelayed SSH reverse tunnel or host backend on lab-maintained VM. |
| Single backend instance fails | All bot interactions fail | systemd `Restart=always` + DB on managed Postgres with PITR; minimum downtime. |
| Students never read warnings | PCs left ON overnight | Idle auto-off sweep guards correctness independent of student cooperation. |
| Master node compromise | Lab PC compromise | Master token rotate; mTLS via Tailscale; master node runs in lab with restricted egress. |
| Scope creep (GPU quotas, persistent homes) | v1 never ships | Strict v1 freeze after SRS sign-off; new asks go to Phase 5+. |

---

## 12. Out of Scope for v1 (parked for Phase 5+)
- GPU job queue / time-slicing / MIG.
- Persistent per-student home directories (NFS or per-PC user home).
- VNC/GUI desktop sessions.
- University SSO integration (Telegram identity is sufficient for v1).
- Calendar invite (.ics) export to students.
- Multi-lab / multi-school support.
- Mobile app beyond Telegram.

---

## 13. Milestone Summary (one-page)
```
M1 (Phase 0): repo, infra, schema, bot hello-world ................ Day 0–3
M2 (Phase 1): book/approve/reject/cancel, notifications ........... Week 1
M3 (Phase 2): real WoL + ephemeral SSH + auto power-off ........... Week 2
M4 (Phase 3): waitlist, reallocation, no-show, master outage ....... Week 3
M5 (Phase 4): dashboards, metrics, runbook, dogfood ............... Week 4
GA (Rollout): superadmin announces to cohort ........................ Week 5
```