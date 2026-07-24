# ReDA Lab Resource Manager — Project Documentation

This folder holds the complete project documentation for the **ReDA Lab Resource Manager** — a Telegram-based system that automates booking, approval, allocation, and power management for the computers in the ReDA School Computer Lab.

## Problem We Solve
The lab today (4 high-spec GPU PCs + ~20 low-spec PCs) is accessed by students over remote SSH. Students submit paper/email request forms, admins manually approve, and students must physically enter the lab to press the power button. This project replaces that with an end-to-end automated Telegram chatbot workflow plus a master node inside the lab that performs Wake-on-LAN and orchestrated SSH actions.

## Documents

| # | Document | Purpose |
|---|---|---|
| 0 | [project-proposal.md](./project-proposal.md) | Project Proposal — executive summary for the Head of ReDA Lab: problem, solution, timeline, cost, risks, and the decisions requested. |
| 1 | [SRS.md](./SRS.md) | Software Requirements Specification — stakeholders, user stories, functional & non-functional requirements, use cases, edge cases, business rules, acceptance criteria, assumptions, glossary. |
| 2 | [SDD.md](./SDD.md) | System Design Document — architecture, technology stack, component breakdown, data flow, deployment topology, security, reliability, MVP vs future phasing, open questions. |
| 3 | [database-erd.md](./database-erd.md) | Database ERD — Mermaid ERD, full per-table column/constraint/index definitions, sample queries, backup/retention, seed data. |
| 4 | [api-specification.md](./api-specification.md) | API Specification — REST endpoints for identity, bookings, PCs, admin, master node, policies, health; auth/RBAC, error model, pagination, rate limits, end-to-end example. |
| 5 | [implementation-roadmap.md](./implementation-roadmap.md) | Implementation Roadmap — phased plan (Phase 0→4 + GA), per-phase work items and acceptance criteria, cross-cutting stream mix, definitions of done, risks & mitigations, milestone summary. |

## Suggested Reading Order
- **Newcomer / PM**: SRS §1–2 → SRS §5 (use cases) → SDD §1–3 → roadmap §2–3.
- **Backend engineer**: SDD §2–3 → ERD §1–2 → API spec → roadmap §4–7.
- **Lab engineer (master node)**: SDD §3.3 + §5.1 → API spec §6 → roadmap §5.
- **Security reviewer**: SDD §6 → ERD §4 (constraints/triggers) → API spec §1.1, §10.

## Document Status
All five documents are at **Draft v1.0** dated **2026-06-30**. Treat them as authoritative for design and implementation; raise changes as a follow-up note in the corresponding file's footer or via a dedicated `CHANGELOG.md` once development starts.

## Open Questions for Stakeholders (see SDD §13)
1. Direct-lab LAN SSH vs Tailscale-mesh access for students?
2. Master node ⇢ PC SSH user model: shared `labadmin` account vs per-PC root key?
3. Is Tailscale or Cloudflare Tunnel permitted by university network policy?
4. Are overnight bookings allowed?
5. Auto-approve of `LOW_SPEC` requests when a PC is immediately available — acceptable default?