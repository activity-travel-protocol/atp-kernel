# AGENTS.md — Activity Travel Protocol SDK

> **Read this file completely before writing any code.**  
> This is the authoritative context document for all AI coding agents working on this repository.  
> Applies to: OpenAI Codex, Cursor Agent, GitHub Copilot Workspace, Gemini Code Assist, and all other AI coding tools.  
> Last updated: April 2026 · v51.0

---

## What This Project Is

The **Activity Travel Protocol** (always full name in prose; `ATP` in code) is an open protocol standard for the global travel industry. It enables discovery, configuration, negotiation, booking, fulfilment, and disruption management of complex travel activities across multiple suppliers, jurisdictions, and AI agents.

**Strategic frame:** ATP is the Win32 API / NT kernel analogy for global travel. The protocol is the interoperability layer. The ATP runtime is the OS. MyAuberge K.K. is Red Hat — a commercial business built on top of an open protocol it governs but does not own.

**Governance:** The Activity Travel Protocol Foundation (一般社団法人アクティビティトラベルプロトコールファウンデーション), an independent Japanese non-profit, governs the protocol under Apache 2.0. MyAuberge K.K. is the Founding Member and commercial IaaS operator.

**Licence:** Apache 2.0

---

## Team

| Person | Role | Scope |
|--------|------|-------|
| Tom Sato | CEO, MyAuberge K.K. / Founding Chair, ATP Foundation | Architecture, strategy, product decisions |
| Mike Ma | CTO, MyAuberge K.K. | Implementation execution — Phase 1 Priority 1 (kernel) and Priority 3 (AI Guest Agent) |

Tom is a former Microsoft Windows/SDK Product Manager. Engage at that level. Do not over-explain fundamentals.

---

## The Reference Implementation — Azusa Journey

Every architectural decision is validated against one real journey:

> Guest books MyAuberge (hotel) + Ponyhouse Farm (activity). Arrives via Azusa Express from Shinjuku. AI agent sends WhatsApp notification when train reaches Shiojiri station. Farm is notified 30 minutes before arrival.

This journey, working end-to-end, **IS** the v1 Activity Travel Protocol implementation. The Azusa Journey 23-step functional spec is the acceptance test. See `ATP_RefImpl_AzusaJourney_v1.docx`.

---

## Architecture — Four Layers (ALL COMPLETE, DO NOT REDESIGN)

| Layer | Description |
|-------|-------------|
| **Layer 1 — Identity & Trust** | Party Registry, Trust Chain, Jurisdiction Compliance, Security Kernel |
| **Layer 2 — Discovery & Capability** | Capability Declarations, Activity Configuration Schema, OCTO Bridge |
| **Layer 3 — Workflow** | Booking Object state machine (11 states, 8 IN_JOURNEY phases), HEM (23 entries), TU-1–TU-6 |
| **Layer 4 — Schema & SDK** | 12 `@atp/` scoped packages, OpenAPI 3.1 (17 endpoints), ATP Condition Expression Syntax v1 |

---

## The 12 SDK Packages

```
@atp/core              — Booking Object runtime, XState v5 state machine, Party Registry
@atp/security          — Security Kernel, Fletcher Embassy, Cedar policy evaluation, mandate validation
@atp/compliance        — Jurisdiction compliance, regulatory_class enforcement
@atp/workflow          — HEM (Human Escalation Manager), TU chain, BOOKING_SUSPENDED
@atp/schema            — OpenAPI 3.1 types, Booking Object schema, all entity schemas
@atp/mcp-server        — ATP MCP Server (8 tools), Windley Loop, NeMo Guardrails rails
@atp/ai-agent          — AI Guest Agent, Context Package delivery, mandate lifecycle
@atp/adapters          — IEventLog interface, PostgresEventLog, adapter swap pattern
@atp/bridge-octo       — OCTO Bridge (Foundation-scaffolded, community-maintained)
@atp/sdk               — Public SDK entry point, developer-facing API surface
@atp/cli               — Developer tooling, conformance testing
@atp/types             — Shared TypeScript types across all packages
```

---

## ATP Kernel — The One-Kernel Model (CLOSED)

**One Node.js process. One PostgreSQL event log. One Cedar evaluation engine. One XState v5 runtime.**

The three reference implementations (Hotel/MyAuberge, Activity Supplier/Ponyhouse, TA/Distributor) are **not** separate kernels. They are **configurations** of one `ATPRuntime`. `regulatory_class` in the Party Registry entry determines which Cedar policy set is evaluated.

| Subsystem | Package | Function |
|-----------|---------|----------|
| Booking Object Runtime | `@atp/core` | Hosts XState v5 instances in memory. Accepts transition requests. Persists to PostgreSQL. |
| Security Kernel | `@atp/security` | Executes **synchronously** on every transition: mandate validation → Cedar evaluation → XState transition → event log write → event emission. **Non-bypassable.** |
| Party Registry | `@atp/core` + `@atp/compliance` | In-memory registry. Ed25519 public keys, `regulatory_class`, trust relationships. Loaded from PostgreSQL on startup. |
| Event Log | `@atp/adapters` (IEventLog) | Append-only `StateTransitionEvent` log. Tier 1: `PostgresEventLog`. Adapter swap = config change, zero code change. |

**Primary spec:** `ATP_KernelSpec_v1.docx` — read this before touching `@atp/core` or `@atp/security`.

---

## Phase 1 Build Sequence — 10 Steps (Mike Ma, owner)

Build in this order. Do not skip steps. The Security Kernel is non-bypassable by design — it cannot be stubbed out.

```
Step 1   PostgreSQL schema — Booking Object event log, Party Registry tables
Step 2   Party Registry — in-memory store, Ed25519 key loading, regulatory_class
Step 3   XState v5 state machine — 11 states, guard conditions, BOOKING_SUSPENDED interrupt
Step 4   Security Kernel skeleton — 5-step execution path, mandate validation stub
Step 5   Cedar policy evaluation — three policy sets (OWN_SUPPLY, FARM_EXPERIENCE, LICENSED_TA)
Step 6   ATP Mandate Model — Ed25519-signed JWT (atp-mandate+jwt), narrowing property enforcement
Step 7   Event log — append-only StateTransitionEvent, PostgresEventLog adapter
Step 8   REST API — 17 OpenAPI 3.1 endpoints, ATPRuntime binding
Step 9   Azusa Journey E2E test — full happy path, booking → in-journey → completed
Step 10  BOOKING_SUSPENDED — cross-cutting interrupt, HEM escalation path
```

**Effort estimate:** 3 weeks (optimistic) to 4–5 weeks (realistic) at 2–3 Claude Code sessions/day.

---

## Critical Implementation Rules — DO NOT VIOLATE

These are CLOSED architectural decisions. Do not reopen, do not work around.

1. **Security Kernel is non-bypassable.** Every state machine transition must pass through the 5-step Security Kernel execution path. No exceptions. No shortcuts for testing.

2. **One kernel, three configurations.** `regulatory_class` in the Party Registry drives Cedar policy set selection. Never fork the kernel for different operator types.

3. **Narrowing property is security-critical.** Delegated mandate ⊆ delegating mandate. Enforced at mandate issuance in `@atp/security`. Mike reads this code personally — do not delegate to an AI agent.

4. **CONFIRMATION hard cap.** AI agents may not confirm bookings autonomously above authority Level 1. This is a protocol invariant.

5. **Windley Loop location.** Runs in the MCP Server (planning gate), not inside the kernel. Security Kernel Cedar evaluation is the enforcement gate. Both are required. They are not redundant.

6. **MCP/A2A boundary.** Booking Object creation event is the boundary. MCP governs Layer 3. A2A governs Layer 2.

7. **Backend first.** ATP backend (this repo) must be complete before the Lovable frontend (booking.myauberge.jp) is rebuilt. Do not rebuild the frontend UI until the API endpoints from Step 8 are stable.

8. **NIM is ATP's inference runtime.** NIM runs inside the ATP container on Red Hat OpenShift. It is not Nokia's platform. Security Kernel is CPU-bound (Cedar + XState v5) — not GPU.

---

## Key Risks — Read Before Starting These Steps

| Risk | Step | Mitigation |
|------|------|------------|
| Cedarling WASM binding in Node 24 | Step 5 | SEC-4 string-match fallback is the safe path. Does not block Steps 6–10. |
| XState v5 BOOKING_SUSPENDED cross-cutting interrupt | Step 10 | Mike reviews nested machine structure explicitly before coding. |
| Narrowing property enforcement at mandate issuance | Step 6 | Security-critical. Mike reads this code personally. |

---

## Guest Agent Communication Channels (CLOSED)

- **WhatsApp** — inbound international guests
- **LINE** — domestic Japanese guests and all supplier-facing notifications
- Do not propose other channels. Do not make channels configurable at the protocol level.

---

## ATP Mandate Model (CLOSED)

- Format: Ed25519-signed JWT (`atp-mandate+jwt`)
- Policy: Cedar partial evaluation
- Binding: `booking_object_id`
- Narrowing property: delegated mandate ⊆ delegating mandate (enforced at issuance)
- Agent architecture: Journey Coordinator (full journey) → Hotel Agent (accommodation phases) → Farm Agent (farm phases only). Path 2 adds Distributor Agent above Hotel Agent.

---

## Booking Object State Machine (CLOSED)

11 states: `DRAFT → PENDING_CONFIRMATION → CONFIRMED → IN_JOURNEY → COMPLETED / CANCELLED / DISPUTED`

8 IN_JOURNEY phases. BOOKING_SUSPENDED is a cross-cutting interrupt that can occur in any state. Full state table: `ATP_KernelSpec_v1.docx`.

---

## Three-Tier Scaling Architecture (CLOSED)

| Tier | Use case | NeMo Guardrails |
|------|----------|-----------------|
| Tier 1 | Development / small operator (MyAuberge) | Absent — prompt layer only |
| Tier 2 | Production | Three normative rails: HEM Escalation, Notification Tone, Scope Boundary |
| Tier 3 | Hyperscale / IaaS | Full rail set |

---

## Regulatory Classes

| Class | Operator type | Cedar policy set |
|-------|--------------|-----------------|
| `OWN_SUPPLY` | MyAuberge sells accommodation + Ponyhouse activities. No TA licence. | OWN_SUPPLY policy set |
| `FARM_EXPERIENCE` | Ponyhouse Farm as independent activity supplier. | FARM_EXPERIENCE policy set |
| `LICENSED_TA` | Registered travel agency sells transport-inclusive packages. | LICENSED_TA policy set |

---

## Current Implementation Priorities

| Priority | Scope | Owner | Blocks |
|----------|-------|-------|--------|
| **1 — Core engine** | Kernel Steps 1–10. Azusa Journey E2E. | Mike Ma | Everything |
| **2 — Ponyhouse integration** | Inter-entity agreement, FARM_EXPERIENCE Capability Declaration, trust config. | Tom Sato (data) / Mike Ma (build) | Blocked on OQ-PONY-7 (Invoice Tax Number) + OQ-PONY-8 (farm address) — Tom to provide |
| **3 — AI Guest Agent** | ATP MCP Server v1, WhatsApp/LINE integration, Context Package delivery, HEM escalation. | Mike Ma | WhatsApp Business API application to Meta — Mike's first-week action |
| **4 — ATP Enabled features** | AI tour guide, weather-adaptive programme, multilingual, guest profile. | TBD | After core engine stable |
| **5 — Admin console** | Booking Object state view, collection dashboard, agent log, disruption management. | Mike Ma | After core engine stable |
| **6 — TA readiness** | Distributor model, TA onboarding UI. | TBD | Build ready; go-live blocked on OQ-JP-1 legal opinion |

---

## Specification Documents

Read these in Claude Code sessions for the relevant build step. They are the authoritative source.

| Document | Read for... |
|----------|-------------|
| `ATP_KernelSpec_v1.docx` | **Step 1–10 — read first.** One-kernel architecture, 4 subsystems, Security Kernel 5-step path, Cedar policy sets, full state machine tables, BOOKING_SUSPENDED spec, ATPRuntime API. |
| `ATP_RefImpl_AzusaJourney_v1.docx` | Step 9 E2E test. 23-step functional spec. Acceptance test definition. |
| `ATP_RefImpl_Ponyhouse_v1.docx` | Priority 2. Ponyhouse Party Registry JSON, Capability Declaration, 10-step build sequence. |
| `ATP_RefImpl_TravelAgency_v1.docx` | Priority 6. Two selling paths, Travel Agency Act, TA Party Registry, Path 2 Booking Object. |
| `ATP_SDK_Architecture_v3.docx` | Layer 4 full SDK architecture. 12 packages, all decisions. |
| `ATP_Layer4_Schema_v1.docx` | OpenAPI 3.1 schema definitions, 17 endpoints. |
| `ATP_CTO_Onboarding_Mike.docx` | Mike's onboarding document. Protocol-to-SDK mapping, blog reading guide, build sequence overview. |

---

## Environment

- **Runtime:** Node.js 24 (check version before starting — Cedarling WASM risk at Step 5)
- **Database:** PostgreSQL (event log + Party Registry persistence)
- **State machine:** XState v5
- **Policy engine:** Cedar (Cedarling WASM binding — see risk above)
- **Inference runtime:** NVIDIA NIM (Tier 2/3 only; not required for kernel Steps 1–10)
- **Container platform:** Red Hat OpenShift (production / AITRAS deployment)
- **Tom's machine:** Windows / PowerShell — write scripts accordingly
- **Mike's machine:** Mac/Linux — shell commands may differ; coordinate before writing automation

---

## OTA Positioning (do not redesign)

Major OTAs (Booking.com, Expedia, Klook, GetYourGuide) are ticketing systems — their data model ends at CONFIRMATION. The ATP Booking Object is the post-confirmation journey coordination layer. ATP wraps OTA reservation engines via OCTO Bridge — it does not replace them.

---

## What ATP Is NOT

- Not a booking UI (that is booking.myauberge.jp, rebuilt by Lovable after this backend)
- Not a payment processor
- Not an OTA
- Not a replacement for existing PMS systems (PMS adapters are `IaaS 3`)
- Not GPU-accelerated at the kernel level (Security Kernel is CPU-bound)

---

## Asking for Architectural Decisions

If you encounter a decision that appears unresolved, check `ARCHITECTURE.md` (Section: Closed Decisions) before asking Tom or Mike to decide. Many questions are already resolved. Do not reopen CLOSED decisions.

If a decision is genuinely open (listed in OQ series), flag it with the OQ ID and do not proceed until Tom or Mike resolves it.

---

*This file is maintained by Tom Sato. Update version number and date when editing.*  
*Spec site: activitytravel.pro · Protocol home: activitytravel.org · GitHub: activity-travel-protocol*
