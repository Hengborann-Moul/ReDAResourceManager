# Project Proposal
## ReDA Lab Resource Manager — Automated Lab PC Booking & Remote Access

**Prepared for:** Head of ReDA Lab
**Prepared by:** Borann Moulheng, ReDA Labs IT
**Date:** 2026-07-23
**Requested decision:** Approval to proceed with a ~5-week v1 build (Phases 0–4) and sign-off on the four policy questions in §9.

---

## 1. Executive Summary

We propose to replace the lab's manual PC booking process — paper/email request forms, manual admin approval, and students physically entering the lab to press the power button — with an **automated Telegram-based booking and power-management system**.

A student's entire journey becomes four steps: **send `/book` → confirm → wait for approval → receive SSH connection details.** The allocated PC powers itself on via Wake-on-LAN, installs the student's SSH key for the duration of their slot, and shuts itself down when the slot ends.

The system requires **no new hardware** beyond one always-on "master node" PC already present in the lab, and runs on a **minimal cloud VM (~$5–10/month)**. Total v1 effort is approximately **6 weeks part-time** for a team of 1–2 backend developers plus a part-time lab engineer. Full design documentation (requirements, architecture, database schema, API specification, and phased roadmap) is already complete and attached to this proposal.

---

## 2. The Problem Today

The ReDA School Computer Lab operates **24 PCs** (4 high-spec GPU machines for ML/LLM workloads, 20 general-purpose) that students access remotely over SSH. The current workflow has real costs:

| Pain point | Consequence |
|---|---|
| Paper/email request forms | Slow turnaround; requests get lost; no queue visibility |
| Manual admin approval and record-keeping | Admin time consumed by routine coordination; no audit trail |
| Students must physically enter the lab to power on a PC | Defeats the purpose of remote access; after-hours access impossible |
| No automatic power-off | PCs left running overnight — wasted electricity and hardware wear |
| Shared/ad-hoc credentials | No per-student accountability on shared machines |
| No usage data | No evidence base for future hardware purchasing decisions |

With demand for the 4 GPU machines growing (LLM coursework, thesis projects), fair and efficient allocation is becoming a daily friction point.

---

## 3. Proposed Solution

A three-part system, fully specified in the attached design documents:

1. **Telegram chatbot** — the only interface students and admins need. Students register once (with their SSH public key), then book with `/book`, choosing category (GPU / general), duration (2/4/8h), and start time. Admins approve or reject with a single button tap in Telegram.

2. **Cloud backend** — a small service that holds all state: bookings, PC inventory, allocation logic, scheduling, notifications, and a complete append-only audit log.

3. **Master node in the lab** — an agent on the existing always-on lab PC that receives commands and performs **Wake-on-LAN power-on**, **ephemeral SSH key installation/removal**, health probes, and **automatic shutdown at slot end**. Crucially, it only ever *dials out* to the backend — **no inbound holes in the lab firewall, no port forwarding, no network policy changes**.

### What the student experiences
```
/book → HIGH_SPEC, 2h, ASAP → admin taps Approve
→ ~3 minutes later: "PC-H1 is ready. ssh labuser@… — slot ends 16:00"
→ 10-minute warning before slot end → automatic shutdown, key removed
```

### Built-in safeguards
- **Ephemeral access:** each student's SSH key is installed only for their slot and removed at the end — no shared lab passwords, per-student accountability.
- **Energy discipline:** automatic shutdown at slot end, plus an idle sweep that powers off any PC left on without an active booking.
- **Fairness:** waitlist with auto-promotion for GPU machines, one active booking per student, and a no-show penalty (3 no-shows in 30 days → 7-day cooldown).
- **Admin control preserved:** every booking requires explicit admin approval (v1 default); admins can force-reclaim any PC at any time with a reason delivered to the student.
- **Full auditability:** every state change and admin action recorded in an immutable audit log, retained ≥ 12 months.

---

## 4. Scope

**In scope (v1):** Telegram registration & booking, admin approve/reject, automatic allocation, WoL power-on, ephemeral SSH keys, automatic power-off, waitlist, no-show handling, force reclaim, audit log, notifications, basic usage reporting, read-only admin dashboard.

**Out of scope (v1, parked for later):** GUI/VNC desktops, persistent per-student home directories, GPU job queues/quotas, university SSO integration, multi-lab support, billing.

---

## 5. Timeline & Milestones

Approximately **5 weeks full-time-equivalent** (≈ 6 weeks part-time), phased so each milestone is a working demo:

| Milestone | Deliverable | Target |
|---|---|---|
| **M1** — Foundations | Repo, database, cloud VM, lab tunnel verified, bot says hello | End of week 1 |
| **M2** — Booking | Students book, admins approve/reject — all in Telegram (no power automation yet) | Week 2 |
| **M3** — Power automation | Approval → PC powers on → student SSHs in → auto shutdown at slot end, on real lab hardware | Week 3 |
| **M4** — Resilience | Waitlist, failure reallocation, no-show handling, master-outage recovery | Week 4 |
| **M5** — Production-ready | Admin dashboard, metrics/alerts, runbook, backups, load-tested | Week 5 |
| **GA** — Rollout | 1-week dogfood with 5 friendly students → announcement to the cohort → 2-week hypercare | Week 6 |

The riskiest assumption — reliable Wake-on-LAN + tunnel connectivity from the lab — is deliberately validated in **week 1** on a single real PC before further investment.

---

## 6. Resources & Cost

| Item | Requirement | Est. cost |
|---|---|---|
| Cloud VM (backend + database) | 1 vCPU / 1 GB is sufficient | ~$5–10 / month |
| Telegram Bot | Free platform | $0 |
| Reverse tunnel (Tailscale free tier or Cloudflare Tunnel) | Free for this scale | $0 |
| Master node | **Existing** always-on lab PC | $0 |
| Lab PCs | Existing; needs WoL enabled in BIOS + SSH server (one-time setup) | $0 |
| People | 1–2 backend devs (part-time) + 1 part-time lab engineer, ~6 weeks | Internal |

**Total recurring cost: ≈ $10/month.** No procurement required.

---

## 7. Benefits & Success Metrics

| Benefit | Measure of success (v1 targets) |
|---|---|
| Faster access for students | Approval → SSH-ready in ≤ 5 minutes (p90) |
| Less admin toil | Booking handled entirely in Telegram; admin action = one button tap |
| Energy savings | Zero PCs left on overnight without a booking (idle sweep) |
| Fair GPU allocation | Waitlist auto-promotion within 60 seconds of a PC freeing up |
| Accountability | 100% of sessions attributable to a named student; audit log retained 12 months |
| Purchasing evidence | Per-PC utilization report available for the next hardware budget cycle |

---

## 8. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| WoL unreliable on some PCs (BIOS/NIC settings) | Bench-tested on one PC in week 1; required BIOS settings documented; wired Ethernet only |
| University network policy blocks the tunnel | Confirm policy before kickoff (see §9); fallback: relayed SSH reverse tunnel or lab-hosted backend |
| Students ignore slot-end warnings | Automation doesn't rely on cooperation: grace period → forced shutdown + idle sweep |
| Scope creep delays launch | Scope frozen at the attached SRS; new requests parked for Phase 5+ |
| Master node fails | Backend keeps accepting bookings ("queued for power-on"); local safety policy shuts down expired slots even if disconnected |

---

## 9. Decisions Needed from You

1. **Approval to proceed** with the v1 build as scoped and phased above.
2. **Network policy:** confirm Tailscale (preferred) or Cloudflare Tunnel is permissible for the lab ↔ cloud link. *(Hard dependency — needed before week 1.)*
3. **SSH access model:** students connect via a Tailscale mesh IP to their allocated PC (recommended), or via the lab LAN/jump host?
4. **After-hours bookings:** allow overnight slots? *(Default: no.)*
5. **Auto-approval for general-purpose PCs:** auto-approve LOW_SPEC requests when a PC is free, to cut admin workload? GPU requests would still always require approval. *(Recommended: yes.)*

---

## 10. Supporting Documents

All design work is complete and available in this folder:

| Document | Contents |
|---|---|
| `SRS.md` | Full requirements: 39 functional requirements, use cases, business rules, acceptance criteria |
| `SDD.md` | Architecture, technology stack, security design, deployment topology |
| `database-erd.md` | Complete database schema (12 tables) |
| `api-specification.md` | REST API specification |
| `implementation-roadmap.md` | Detailed phase-by-phase build plan with acceptance criteria |
| `project-blueprint.drawio` | 9-page visual blueprint: architecture, booking lifecycle, ERD, API map, roadmap, and Telegram UI mockups for every booking state |

---

## 11. Recommended Next Steps

1. 30-minute walkthrough of this proposal and the visual blueprint (we can present the draw.io pages).
2. Your decisions on §9 (particularly the network-policy question).
3. Kickoff: Phase 0 starts the following Monday; first end-to-end demo on real hardware at **M3 (week 3)**.
