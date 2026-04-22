# ATP Kernel

Reference implementation of the **Activity Travel Protocol** — an open protocol standard for the global travel industry.

**Specification site:** [activitytravel.pro](https://activitytravel.pro)  
**Protocol home:** [activitytravel.org](https://activitytravel.org)  
**Licence:** Apache 2.0  
**Governance:** [Activity Travel Protocol Foundation](https://activitytravel.org/governance) (一般社団法人アクティビティトラベルプロトコールファウンデーション)

---

## What This Is

The Activity Travel Protocol is a runtime platform — a Travel Operating System — that manages the full lifecycle of a booking as a first-class runtime entity, with policy enforcement, trust chain construction, duty of care tracking, and AI agent participation built into the protocol itself.

This repository is the reference implementation of that runtime: the ATP Kernel.

The kernel is a single Node.js process that hosts the Booking Object state machine, Security Kernel, Party Registry, and event log. It is the canonical implementation against which the protocol specification is validated.

---

## Protocol Architecture

The Activity Travel Protocol is defined across four layers. All four layers are complete and published at [activitytravel.pro](https://activitytravel.pro).

| Layer | Name | Description |
|-------|------|-------------|
| **Layer 1** | Identity & Trust | Party Registry, Trust Chain, Security Kernel, Jurisdiction Compliance |
| **Layer 2** | Discovery & Capability | Capability Declarations, Activity Configuration Schema, OCTO Bridge |
| **Layer 3** | Workflow | Booking Object state machine, Human Escalation Manager, disruption management |
| **Layer 4** | Schema & SDK | 12 `@atp/` scoped packages, OpenAPI 3.1, ATP Condition Expression Syntax |

---

## Kernel Architecture

The kernel is one runtime. One Node.js process. One PostgreSQL event log. One Cedar evaluation engine. One XState v5 state machine.

Multiple operator types (hotel, activity supplier, licensed travel agency) are configurations of one `ATPRuntime` — not separate kernels. The `regulatory_class` field in the Party Registry entry determines which Cedar policy set is evaluated.

### Core Subsystems

| Subsystem | Package | Description |
|-----------|---------|-------------|
| Booking Object Runtime | `@atp/core` | XState v5 state machine instances. Accepts transition requests. Persists snapshots to PostgreSQL. |
| Security Kernel | `@atp/security` | Executes synchronously on every transition. Non-bypassable. |
| Party Registry | `@atp/core` | In-memory registry of all parties. Ed25519 public keys, trust relationships, regulatory class. |
| Event Log | `@atp/adapters` | Append-only `StateTransitionEvent` log via adapter interface. |

### SDK Packages

```
@atp/core              — Booking Object runtime, XState v5 state machine, Party Registry
@atp/security          — Security Kernel, Fletcher Embassy, Cedar policy evaluation
@atp/compliance        — Jurisdiction compliance, regulatory_class enforcement
@atp/workflow          — Human Escalation Manager, TU chain, BOOKING_SUSPENDED
@atp/schema            — OpenAPI 3.1 types, Booking Object schema, entity schemas
@atp/mcp-server        — ATP MCP Server (8 tools), Windley Loop, NeMo Guardrails
@atp/ai-agent          — AI Guest Agent, Context Package delivery, mandate lifecycle
@atp/adapters          — IEventLog interface, PostgresEventLog, adapter pattern
@atp/bridge-octo       — OCTO Bridge (Foundation-scaffolded, community-maintained)
@atp/sdk               — Public SDK entry point, developer-facing API surface
@atp/cli               — Developer tooling, conformance testing
@atp/types             — Shared TypeScript types
```

---

## Reference Implementation — Azusa Journey

The acceptance test for the v1 kernel is the **Azusa Journey**:

> A guest books MyAuberge (hotel) and Ponyhouse Farm (activity). They arrive via the Azusa Express from Shinjuku. The AI agent sends a WhatsApp notification when the train reaches Shiojiri station. The farm is notified 30 minutes before arrival.

This end-to-end journey — covering trust establishment, capability declaration, booking state transitions, AI agent mandate enforcement, and disruption management — is the definitive test that the kernel conforms to the protocol specification.

---

## Status

| Component | Status |
|-----------|--------|
| Protocol specification (all 4 layers) | ✅ Complete — published at activitytravel.pro |
| ATP Kernel (`@atp/core`, `@atp/security`) | 🔨 In development |
| ATP MCP Server (`@atp/mcp-server`) | 🔨 In development |
| AI Guest Agent (`@atp/ai-agent`) | 🔨 In development |
| OCTO Bridge (`@atp/bridge-octo`) | 📋 Planned |
| Public SDK (`@atp/sdk`) | 📋 Planned |

---

## Getting Started

### Prerequisites

- Node.js 24+
- PostgreSQL 16+
- Read [`CLAUDE.md`](./CLAUDE.md) before contributing — it contains the full architectural context, build sequence, and closed decisions that govern this implementation.

### Protocol Specification

The specification is the authoritative reference for all implementation decisions. Working drafts and published specifications are at [activitytravel.pro/working-drafts](https://activitytravel.pro/working-drafts/).

---

## Key Protocol Invariants

These are non-negotiable protocol requirements enforced by this implementation:

- The Security Kernel executes on every Booking Object state transition without exception.
- AI agents may not confirm bookings autonomously above authority Level 1.
- Delegated mandate authority cannot exceed the authority of the delegating party (narrowing property).
- The Booking Object is the fundamental runtime entity — all state and event history is derived from it.

---

## Contributing

Contributions to the reference implementation are welcome. Please read the [Contributor Licence Agreement](https://activitytravel.pro/working-drafts/governance/cla/) before submitting a pull request.

For protocol specification contributions, see [activity-travel-protocol/protocol-spec](https://github.com/activity-travel-protocol/protocol-spec).

For Foundation membership, visit [activitytravel.org](https://activitytravel.org).

---

## Licence

Apache 2.0. See [LICENSE](./LICENSE).

---

## Governance

The Activity Travel Protocol is governed by the Activity Travel Protocol Foundation (一般社団法人アクティビティトラベルプロトコールファウンデーション), an independent standards body constituted under Japanese law.

MyAuberge K.K. is the Founding Member. The Foundation governs the protocol specification. MyAuberge K.K. operates the commercial IaaS implementation.

[activitytravel.org/governance](https://activitytravel.org/governance)
