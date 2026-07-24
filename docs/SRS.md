# Software Requirements Specification (SRS)
## ReDA Lab Resource Manager — Telegram-Based Lab PC Booking & Power Automation

**Document Status:** Draft v1.0
**Last Updated:** 2026-06-30
**Owner:** ReDA Labs IT

---

## 1. Introduction

### 1.1 Purpose
This document specifies the functional, non-functional, and behavioral requirements for the **ReDA Lab Resource Manager** — a system that automates the booking, approval, allocation, and power management of computers in the ReDA School Computer Lab through a Telegram chatbot interface. It is the authoritative requirements source for design, implementation, and acceptance testing.

### 1.2 Scope
The system replaces the current manual flow (paper/email request forms + admin approval + physical power-on by the student) with an end-to-end automated workflow. The system covers:

- Student booking requests via Telegram
- Admin approval/rejection via Telegram
- Automatic allocation of a lab PC matching the requested category (high-spec / low-spec)
- Automatic power-on of the allocated PC via Wake-on-LAN (WoL) issued from a master node inside the lab network
- Distribution of SSH connection details to the student
- Automatic power-off (or reclaim) at the end of the booked slot
- Audit trail, notifications, and basic reporting

**Out of scope (v1):** GUI desktop environment per student, persistent home directories per student, GPU job scheduling/quotas, billing/payment, integration with the university SSO/LMS. These are listed as future enhancements.

### 1.3 Definitions, Acronyms, and Glossary

| Term | Definition |
|---|---|
| **PC Category** | Classification of a lab PC: `HIGH_SPEC` (GPU-capable, for LLM/ML workloads) or `LOW_SPEC` (general purpose). |
| **Booking / Request** | A student's formal ask to use a lab PC for a specific slot and purpose. |
| **Allocation** | The act of assigning a specific PC to an approved booking. |
| **Slot** | A bounded time window (start datetime → end datetime) for which a PC is allocated to a student. |
| **Master Node** | The always-on PC inside the lab that runs the controller service. It receives commands from the backend and performs WoL, SSH health probes, and remote shutdown. |
| **WoL** | Wake-on-LAN — network protocol to power on a PC by sending a "magic packet" to its NIC. |
| **No-show** | A booking where the allocated PC was powered on but the student never established an SSH session within the grace window. |
| **Force Reclaim** | Admin-initiated early termination of an active booking, including a graceful (then forced) shutdown of the PC. |
| **Bot Service** | The component that talks to Telegram. |
| **Core Backend** | The component holding business logic, REST API, and the scheduler. |
| **Master Node Controller** | The agent running on the master node PC inside the lab. |

### 1.4 References
- IEEE Std 830-1998 (Recommended Practice for SRS)
- Telegram Bot API: https://core.telegram.org/bots/api
- Wake-on-LAN: RFC 6171 (magic packet)
- See also: `SDD.md`, `database-erd.md`, `api-specification.md`, `implementation-roadmap.md`

### 1.5 Overview
Section 2 describes the overall description and product context. Section 3 lists functional requirements. Section 4 lists non-functional requirements. Section 5 contains use cases. Section 6 covers edge cases and business rules. Section 7 lists acceptance criteria. Section 8 lists assumptions and dependencies.

---

## 2. Overall Description

### 2.1 Product Context
The system sits between three parties: students (requesters), lab administrators (approvers/operators), and the lab hardware (PCs + master node). The student and admin interact exclusively through a Telegram chatbot. The backend (cloud-hosted) holds business state and orchestrates the master node, which in turn controls individual lab PCs over the lab LAN.

```
[Students] --(Telegram)--> [Bot Service] --> [Core Backend] <--(HTTPS long-poll/WS)--> [Master Node Controller]
[Admins]  --(Telegram)--> [Bot Service] --> [Core Backend]                                            |
                                      \----> [Database]                                              +--> WoL/SSH/ping to each Lab PC
[Admin Dashboard Web] --> [Core Backend REST API]
```

### 2.2 User Classes and Characteristics

| User Class | Description | Characteristics |
|---|---|---|
| **Student** | Any registered lab user who needs remote access to a lab PC. | Large fluctuating population; only needs Telegram; technically competent (SSH use). |
| **Lab Administrator** | Lab staff responsible for approving requests and managing inventory. | Small group (~1–3 people); needs quick decision UI; not necessarily technical. |
| **Super Administrator** | Maintains admin roster, PC inventory, and policy config. | 1–2 trusted people; configures the system. |
| **System (Automated Actor)** | The backend + master node automation enforcing scheduled power-on/off, reallocation, and notifications. | Not a human; must be deterministic and observable. |

### 2.3 Operating Environment
- Telegram client on user mobile/desktop devices
- Cloud-hosted backend (Linux, x86_64, containerized)
- Lab master node: always-on Linux PC inside the lab LAN, behind NAT/firewall
- Up to 24 lab PCs (4 high-spec with GPUs, 20 low-spec), each WoL-capable, each running an SSH server

### 2.4 Design Constraints
- The lab network is behind NAT; the cloud backend cannot directly reach a lab PC. All commands must egress from the master node.
- Telegram is the only required student/admin client.
- Lab PCs must support WoL in their BIOS/UEFI and have an SSH server installed.

### 2.5 Assumptions and Dependencies
See Section 8.

---

## 3. Functional Requirements

Requirements are numbered for traceability. Each requirement is tagged with priority: **[M]** = must (MVP), **[S]** = should, **[C]** = could.

### 3.1 Identity & Registration
- **FR-1 [M]** The system shall identify Telegram users by their Telegram User ID.
- **FR-2 [M]** A student must register once before submitting a request. Registration captures: full name, student ID number, course/lab group, preferred contact. The Bot shall deny requests from unregistered users.
- **FR-3 [M]** Registration creates an account with role `STUDENT` and status `PENDING` until a superadmin approves it (or auto-approve is enabled in config).
- **FR-4 [M]** Superadmin shall be able to promote/demote users between `STUDENT`, `ADMIN`, `SUPERADMIN` roles.
- **FR-5 [M]** Superadmin shall be able to block/unblock a user. Blocked users cannot submit new requests.

### 3.2 PC Inventory Management
- **FR-6 [M]** Each lab PC shall be registered with: hostname, MAC address (for WoL), LAN IP, category (`HIGH_SPEC`/`LOW_SPEC`), specs summary (CPU/RAM/GPU description), status (`AVAILABLE`/`IN_USE`/`MAINTENANCE`/`OFFLINE`), and location label.
- **FR-7 [M]** Admins shall be able to add/edit/disable PC records.
- **FR-8 [M]** A PC in `MAINTENANCE` status shall be excluded from allocation.
- **FR-9 [S]** The master node shall periodically probe each PC and update its live status (`ON`/`OFF`/`UNREACHABLE`).

### 3.3 Booking Request Lifecycle
- **FR-10 [M]** A student shall be able to submit a booking request specifying:
  - PC category (`HIGH_SPEC` or `LOW_SPEC`)
  - Workload description (free-form, 5–300 characters)
  - Desired slot duration (configurable options; default 2 / 4 / 8 hours)
  - Desired start time (ASAP or a future slot within the configured advance window)
- **FR-11 [M]** On submission, the system shall create a `Booking` in state `PENDING` and notify all admins via Telegram with an inline Approve/Reject keyboard.
- **FR-12 [M]** Admin shall be able to approve or reject a pending booking with an optional reason (free text or quick-pick reasons).
- **FR-13 [M]** On reject, the student shall be notified with the reason; the booking transitions to `REJECTED`.
- **FR-14 [M]** On approve, the system shall run the allocation algorithm (see FR-15), power on the PC (see FR-17), and notify the student with SSH details (see FR-20). Booking transitions to `APPROVED` then `ACTIVE` once the PC confirms ON.
- **FR-15 [M]** Allocation algorithm: among PCs of the requested category with status `AVAILABLE` and not overlapping any existing confirmed booking for the slot, pick the one with the fewest recent allocations (round-robin-ish load balancing). If none available, attempt alternatives per FR-19.
- **FR-16 [M]** A student shall be able to cancel their own pending or active booking. Cancellation of an active booking triggers a shutdown of the PC.
- **FR-17 [M]** On approval, the backend shall issue a `POWER_ON` command to the master node for the allocated PC's MAC. The master node shall send the WoL magic packet and then poll (ping/SSH-probe) until the PC is reachable or until a timeout (default 5 minutes) is hit.
- **FR-18 [M]** If the allocated PC does not power on within the timeout, the system shall retry allocation on a different PC (up to a configurable retry count, default 2) before notifying the admin that the request needs manual intervention.
- **FR-19 [M]** If no matching-category PC is available when a high-spec request is approved, the system shall NOT silently substitute a low-spec PC. It shall place the booking in `WAITLIST` and notify the student; when a slot frees up, auto-promote the oldest eligible waitlisted booking.
- **FR-20 [M]** On successful power-on, the system shall notify the student with: PC label, LAN IP, SSH username, slot end time, and a "How to connect" snippet.
- **FR-21 [M]** SSH access shall be per-booking ephemeral: the master node adds the student's registered public SSH key to the PC's `authorized_keys` for the slot and removes it on slot end.

### 3.4 Power & Lifecycle Management
- **FR-22 [M]** At configured offset (default 10 minutes) before slot end, the system shall send a warning notification to the student.
- **FR-23 [M]** At slot end, the system shall attempt a graceful shutdown over SSH (`sudo shutdown -h now`). If no SSH session is detected and the PC is idle (CPU < threshold for > 5 min), escalate to forced power-off via the master node.
- **FR-24 [M]** If the student still has an active SSH session at slot end, the system shall grant a configurable grace period (default 15 minutes) with a warning, then force shutdown. Admins may grant an extension (FR-30).
- **FR-25 [M]** A daily scheduled job shall power off any PC that has been ON without an active booking for more than the idle threshold (default 30 minutes), to recover from leaked states.
- **FR-26 [M]** No-show handling: if a booking is active but no SSH session from the student's key has been established within a grace window (default 20 minutes) after power-on, the system shall release the booking and power off the PC.

### 3.5 Admin Operations
- **FR-27 [M]** Admin shall be able to list all current bookings (with filter by status/category).
- **FR-28 [M]** Admin shall be able to force-reclaim a PC from an active booking, with a mandatory public reason visible to the student.
- **FR-29 [M]** Admin shall be able to manually power-off or restart any PC.
- **FR-30 [M]** Admin shall be able to grant a one-time slot extension to an active booking (configurable max extension, default 2 hours).
- **FR-31 [M]** Admin shall be able to set a PC into `MAINTENANCE` status (excludes from allocation; does not auto power off existing bookings).
- **FR-32 [S]** Admin shall be able to broadcast an announcement to all registered students via Telegram.
- **FR-33 [S]** Superadmin shall be able to view an audit log of all admin actions.

### 3.6 Notifications
- **FR-34 [M]** The system shall send Telegram notifications for: booking submitted (admins), booking approved/rejected (student), PC powered on + SSH details (student), slot-end warning (student), slot-end grace (student), no-show release (student), force reclaim (student), master node offline alert (admins).
- **FR-35 [M]** Notifications shall be idempotent: a duplicate-send shall not corrupt booking state.
- **FR-36 [S]** Reminders for upcoming scheduled bookings (default 1 hour before scheduled start).

### 3.7 Audit & Reporting
- **FR-37 [M]** Every state transition of a Booking and every admin action shall be appended to an immutable `AuditLog` with actor, timestamp, action, before/after where applicable.
- **FR-38 [S]** Admins shall be able to generate a usage report for a date range: total bookings, approval rate, no-show rate, per-PC utilization hours, per-student booking counts.

### 3.8 Admin Web Dashboard (read-only for v1)
- **FR-39 [S]** A minimal read-only web dashboard at the backend's REST API shall expose the same data as the admin views in Telegram (current bookings, PC status, audit log), authenticated via an admin-token or Telegram login widget.

---

## 4. Non-Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| **NFR-1** | Performance | Booking submission → admin notification latency ≤ 5 seconds (p95), excluding Telegram platform latency. |
| **NFR-2** | Performance | Approve→PC-powered-on (PC reachable over SSH) ≤ 5 minutes (p90) for a healthy cold-boot PC. |
| **NFR-3** | Availability | Backend uptime ≥ 99% measured monthly; gracefull degradation: if master node is offline, booking/approval still works and shows "queued for power-on when master returns". |
| **NFR-4** | Reliability | The system shall not lose a booking during a backend restart (durable DB-backed state). |
| **NFR-5** | Security | All admin actions require role check; non-admins cannot approve/reject or manage inventory. |
| **NFR-6** | Security | All backend ↔ master node traffic shall be TLS-encrypted and authenticated with a shared secret/mTLS. |
| **NFR-7** | Security | Student SSH credentials are per-booking ephemeral keys (no shared lab password in v1); keys are revoked at slot end. |
| **NFR-8** | Security | Telegram bot shall verify `allowed_updates` and ignore messages from unregistered users with a help reply, never leaking internal IDs to unregistered users. |
| **NFR-9** | Scalability | The system shall support up to 500 registered students and ~30 lab PCs without architectural change. |
| **NFR-10** | Usability | Student-facing bot shall require ≤ 4 message exchanges to submit a booking. |
| **NFR-11** | Maintainability | Code, schema, and config shall be version-controlled; migrations are reversible. |
| **NFR-12** | Observability | Structured JSON logs with booking_id, pc_id, actor; metrics endpoint for uptime, queue depth, master node heartbeat, power-on success rate. |
| **NFR-13** | Localization | Bot copy shall be externalized to a locale file; English shipped first; pluggable for other languages later. |
| **NFR-14** | Privacy | Workload descriptions are visible to admins but not to other students. |
| **NFR-15** | Auditability | Audit logs retained for ≥ 12 months. |
| **NFR-16** | Portability | Backend runs as a Docker container; master node controller runs as a systemd service. |

---

## 5. Use Cases

### UC-1: Submit Booking Request
| | |
|---|---|
| **Actor** | Student |
| **Precondition** | Student is registered and not blocked. |
| **Main Flow** | 1. Student sends `/book` to bot. 2. Bot asks category (HIGH_SPEC/LOW_SPEC). 3. Bot asks workload description. 4. Bot asks duration/start (defaults: 2h, ASAP). 5. Bot shows confirmation summary. 6. Student confirms. 7. Booking created as `PENDING`. 8. Admins notified. 9. Student receives confirmation + pending ID. |
| **Alternative Flow** | A1: Student not registered → bot replies with registration prompt. A2: Student already has an active or pending booking → bot asks to cancel first. A3: No free slot in advance window → bot informs and asks to pick another time. |
| **Postcondition** | A `PENDING` Booking exists; admins received a Telegram message with inline Approve/Reject. |

### UC-2: Approve / Reject Request
| **Actor** | Admin |
| **Precondition** | A `PENDING` booking exists. |
| **Main Flow** | 1. Admin taps Approve or Reject on the inline keyboard. 2. If Reject, bot asks for an optional reason. 3. Booking transitions to `REJECTED` (with reason) or `APPROVED`. 4. On approve, allocation runs (UC-3). |
| **Alternative Flow** | A1: Booking already decided (race) → bot replies "already handled by &lt;admin&gt;". A2: Admin not authorized → ignored + logged. |
| **Postcondition** | Booking is in `REJECTED` or `APPROVED`/`ACTIVE` state. |

### UC-3: Auto Power-On After Approval
| **Actor** | System |
| **Precondition** | Booking is `APPROVED`; master node is online. |
| **Main Flow** | 1. Backend selects a candidate PC via allocation algorithm. 2. Backend issues `POWER_ON` to master node with PC MAC. 3. Master node sends WoL magic packet, then polls SSH/ping. 4. On reachability, master node pushes student's SSH pubkey to `authorized_keys`. 5. Master node reports `POWERED_ON`. 6. Backend transitions booking → `ACTIVE`, notifies student with SSH details. |
| **Alternative Flow** | A1: PC unreachable within 5 min → retry on a different PC up to N times; if all fail, mark booking `NEEDS_MANUAL` and notify admin. A2: Master node offline → mark booking `POWER_ON_QUEUED`; auto-retry when master node heartbeat resumes. |

### UC-4: Auto Power-Off at Slot End
| **Actor** | System |
| **Main Flow** | 1. At T-start − 10 min: warning sent to student. 2. At slot end: try graceful SSH shutdown. 3. If session active: grant 15-min grace (config), warn. 4. After grace or no session: force shutdown (master node power-off) and remove ephemeral key. 5. Booking → `COMPLETED`. 6. PC returns to `AVAILABLE`. |

### UC-5: PC Failure & Reallocation
| **Actor** | System |
| **Main Flow** | 1. During `POWER_ON`, PC unreachable past timeout. 2. Retry Nb times on alternative PCs. 3. If reallocation succeeds: continue UC-3 path with new PC. 4. If all retries fail: booking → `NEEDS_MANUAL`, admin notified, student notified "pending manual setup". |
| **Postcondition** | Either the student is allocated a working PC, or the admin is taking over. |

### UC-6: Cancel Own Booking
| **Actor** | Student |
| **Main Flow** | 1. Student sends `/mybookings`. 2. Bot inline-keyboards active/pending bookings with "Cancel" button. 3. Student taps Cancel. 4. Confirmation prompt. 5. Booking → `CANCELLED`. 6. If active, issue shutdown via master node and remove ephemeral key. 7. Confirm to student. |

### UC-7: Admin Force Reclaim a PC
| **Actor** | Admin |
| **Main Flow** | 1. Admin sends `/reclaim <pc-label>` or taps from `/status`. 2. Bot demands a public reason (will be shown to student). 3. Booking → `RECLAIMED`. 4. Graceful shutdown → force after grace. 5. Student notified with reason + slot/PC released. 6. PC → `AVAILABLE` (or `MAINTENANCE` if requested). |

### UC-8: Admin Manages PC Inventory
| **Actor** | Superadmin |
| **Main Flow** | 1. `/pc add <host> <mac> <ip> <category>`. 2. Validate format. 3. Insert PC record. 4. `/pc disable <host>` for maintenance. 5. Changes logged in AuditLog. |

---

## 6. Edge Cases, Business Rules, and Constraints

### 6.1 Edge Cases
- **EC-1:** All high-spec PCs booked when a `HIGH_SPEC` request is submitted → booking is `WAITLISTED`, not silently downgraded. Configurable: `allow_low_spec_substitute_on_waitlist = false` (default).
- **EC-2:** Student no-shows (never SSH'd within 20 min) → release booking, shut down PC, mark as `NO_SHOW`, apply no-show penalty (see BR-4).
- **EC-3:** Student still SSH'd at slot end → grant 15-min grace with warning, then forced shutdown. Admin can extend.
- **EC-4:** PC fails to power on → reallocation per UC-5.
- **EC-5:** Telegram bot / Telegram API outage → backend keeps state, retries push notifications with exponential backoff up to 24h; missed notifications are summarized on recovery.
- **EC-6:** Master node loses connectivity to backend → master node continues running local cron-based safety (power-off any PC without a known active slot), and reconnects via long-poll; backend marks `master_node_status = OFFLINE` and queues commands.
- **EC-7:** Lab power/network outage → all PCs become `UNREACHABLE`; active bookings are preserved with state `INTERRUPTED` and admins notified; on recovery, admins can resume or release.
- **EC-8:** Duplicate/concurrent requests from same student → only one `PENDING`/`ACTIVE` per student at a time (configurable max active bookings per student, default 1).
- **EC-9:** Booking extends beyond lab operating hours → reject with reason, or allow if `allow_after_hours` is configured true.
- **EC-10:** Student edits or rotates their registered SSH key between booking and slot → ephemeral key for the booking is captured at approval time; rotation does not affect an active booking.

### 6.2 Business Rules
| ID | Rule |
|---|---|
| **BR-1** | Slot duration options: 2h, 4h, 8h (admin-configurable list). |
| **BR-2** | Advance booking window: up to 7 days in the future (configurable). |
| **BR-3** | Max concurrent bookings per student: 1 (configurable). |
| **BR-4** | No-show penalty: 3 no-shows in 30 days → 7-day submission cooldown (configurable). |
| **BR-5** | High-spec bookings require admin approval (default) — superadmin may set `auto_approve_low_spec = true` to auto-approve `LOW_SPEC` requests if a PC is immediately available. |
| **BR-6** | Force-reclaim requires a reason that is delivered to the student. |
| **BR-7** | Ephemeral SSH access only — no shared lab password issued to students. |

---

## 7. Acceptance Criteria (Headline Capability)

**Capability:** A student's approved booking results in a lab PC automatically powering on, and the student receiving SSH connection details.

**Given** the student `S1` is registered and not blocked, and an `HIGH_SPEC` PC `PC-H1` with status `AVAILABLE` and a healthy master node `M`,

**When** `S1` submits a booking for `HIGH_SPEC`, 2 hours, ASAP, and the admin approves,

**Then**
1. Within 5 seconds, all admins receive a Telegram notification of the pending booking. *(NFR-1)*
2. Within 10 seconds of admin approval, the backend selects `PC-H1` (or another suitable `HIGH_SPEC` PC) and issues a `POWER_ON` to `M`. *(FR-15, FR-17)*
3. Within 5 minutes (p90), `M` confirms `PC-H1` is reachable via SSH. *(NFR-2, UC-3)*
4. `M` adds `S1`'s registered SSH public key to `PC-H1`'s `authorized_keys`. *(FR-21)*
5. `S1` receives a Telegram message containing: PC label, LAN IP, SSH username, slot start, slot end, and a "how to connect" snippet. *(FR-20)*
6. `S1` can SSH into `PC-H1` using their private key within the slot. *(FR-21)*
7. The Booking state is `ACTIVE`. *(FR-14)*
8. An `AuditLog` entry is written for the approval, allocation, and power-on event. *(FR-37)*
9. At `slot_end - 10 min`, `S1` receives a warning. *(FR-22)*
10. At `slot_end`, the system gracefully shuts down `PC-H1` (or grants grace if `S1` is still connected) and removes the ephemeral key. `PC-H1` returns to `AVAILABLE`. *(FR-23, FR-21)*

---

## 8. Assumptions and Dependencies

### 8.1 Assumptions
- The lab has one always-on master node PC running Linux, connected to the lab LAN and to the internet via the lab's router.
- All lab PCs have WoL-capable NICs and WoL is enabled in BIOS/UEFI and OS driver.
- Each lab PC runs an SSH server and is reachable from the master node on the lab LAN.
- The cloud backend has a stable outbound internet connection (Telegram + reachable ingress for admin web).
- The university permits a reverse tunnel/cloud relay between the lab and the backend (or Tailscale is allowed).

### 8.2 Dependencies
- Telegram Bot API (and a valid bot token from `@BotFather`).
- A PostgreSQL database (managed or self-hosted).
- A cloud host (e.g. small VM) for the backend.
- An optional domain name + TLS cert for the admin web endpoint (Let's Encrypt acceptable).
- Tailscale, Cloudflare Tunnel, or an equivalent reverse-tunnel mechanism for master node ↔ backend connectivity.
- `wakeonlan` / raw UDP socket capability on the master node for magic packets.
- `paramiko` (or `asyncssh`) on the master node for SSH-driven actions.

---

## 9. Future Enhancements (Out of v1 Scope)
- GUI/VNC desktop environment per student session.
- Persistent per-student home directories on lab PCs or NFS-shared.
- GPU time-slicing / MIG / job queue for high-spec PCs.
- Idle-aware auto-suspend/resume.
- Integration with the university SSO for student identity verification.
- Slots marketplace / swap requests between students.
- Real-time dashboards (Grafana) on top of the metrics endpoint.
- Multi-lab / multi-site support.