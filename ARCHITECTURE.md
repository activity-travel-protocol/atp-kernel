# ATP Kernel — Architecture

> This document explains the **why** behind every major design decision in the ATP Kernel.  
> For the **what** and **how**, see `CLAUDE.md` (build sequence, rules, package map).  
> For the protocol specification, see [activitytravel.pro](https://activitytravel.pro).  
> Last updated: April 2026 · v51.0

---

## 1. The Core Idea — The Booking Object as Runtime Entity

Most travel systems treat a booking as a database record. A row is created at confirmation and updated as state changes. The booking is passive data.

The Activity Travel Protocol treats a booking as a **live runtime entity** — a first-class process with its own state machine, policy enforcement, event history, and AI agent participation rights. The booking is active infrastructure.

This distinction drives every architectural decision in the kernel. The Booking Object is not stored in a table and queried. It is **hosted in memory as a running state machine**, with every transition enforced by the Security Kernel before it is allowed to proceed.

The analogy from Tom Sato's Microsoft background: this is the Win32 process model applied to travel bookings. A booking is a process. The ATP Kernel is the OS that schedules, enforces, and persists it.

---

## 2. One Kernel, Three Configurations

Early in the design process, a natural instinct was to build separate implementations for each operator type: one kernel for hotels, one for activity suppliers, one for licensed travel agencies. This was rejected.

**Why one kernel:**

The protocol invariants — the Security Kernel execution path, the mandate model, the event log structure, the Booking Object state machine — are identical across all operator types. What differs is **policy**: which transitions are permitted, which Cedar rules are evaluated, which regulatory constraints apply.

The `regulatory_class` field in a Party Registry entry is the switch. It selects the Cedar policy set at runtime. Three operator types become three Cedar policy sets:

| `regulatory_class` | Operator type | Cedar policy set |
|--------------------|--------------|-----------------|
| `OWN_SUPPLY` | Hotel sells own accommodation + own activities | No transport intermediation rules |
| `FARM_EXPERIENCE` | Independent activity supplier | Activity-specific duty of care |
| `LICENSED_TA` | Registered travel agency with transport | Japan Travel Agency Act compliance |

One codebase. Three configurations. This is not a shortcut — it is the correct architecture. A licensed TA and a farm activity supplier share 95% of the same protocol machinery. Forking the kernel for policy differences would make both implementations harder to maintain and harder to certify as conformant.

---

## 3. The Security Kernel — Why Non-Bypassable

The Security Kernel executes a 5-step path on every Booking Object state transition:

```
1. Mandate validation     — is the requesting agent authorised to request this transition?
2. Cedar evaluation       — does the transition comply with all applicable Cedar policies?
3. XState transition      — execute the state machine transition
4. Event log write        — append StateTransitionEvent to PostgreSQL (append-only)
5. Event emission         — notify subscribers
```

Steps 1 and 2 are **guards**. If either fails, the transition does not proceed. Steps 3, 4, and 5 are atomic — they either all succeed or the transition is rolled back.

**Why non-bypassable:**

The protocol's core promise to operators, regulators, and travellers is that every state transition was authorised, policy-compliant, and permanently recorded. If any code path can bypass the Security Kernel — for performance, for testing convenience, for a quick fix — that promise is broken for every booking processed through that path.

This is not theoretical. The scenarios where someone would want to bypass the Security Kernel are exactly the scenarios where bypassing it is most dangerous: bulk operations under load, emergency overrides during disruption, developer shortcuts in staging environments that quietly make it to production.

The kernel has no bypass mode. There is no `unsafe_transition()`. There is no `skip_policy_check` flag. Tests use the real Security Kernel against test fixtures. Production and test are the same code path.

**The Windley Loop is separate:**

The Windley Loop (mandatory Cedar partial evaluation before an AI agent begins planning a session) runs in the **MCP Server**, not the kernel. This is the planning gate — it prevents an agent from spending 10 minutes planning a booking sequence that Cedar would reject at execution time.

The Security Kernel Cedar evaluation is the **enforcement gate** — it cannot be fooled by a planning-time evaluation, because state changes between planning and execution. Both gates are required. They are not redundant.

---

## 4. XState v5 — Why a State Machine

The Booking Object has 11 states and 8 IN_JOURNEY phases. The legal transitions between states are not arbitrary — they are protocol requirements. A booking cannot move from `DRAFT` to `IN_JOURNEY` without passing through `CONFIRMED`. `BOOKING_SUSPENDED` can interrupt any state. `CANCELLED` is terminal.

Encoding this in ad-hoc conditional logic (a series of `if booking.status === 'confirmed'` checks) produces code that is correct until it isn't. State transition bugs in travel systems are expensive: double-confirmed bookings, revenue that was never collected, guests who were confirmed but whose suppliers were never notified.

XState v5 makes illegal transitions impossible at the type level. The state machine definition is the authoritative record of what transitions are permitted. A developer cannot accidentally write `transitionTo('IN_JOURNEY')` from `DRAFT` — XState rejects it.

**BOOKING_SUSPENDED as cross-cutting interrupt:**

`BOOKING_SUSPENDED` is not a normal state — it is an interrupt that can occur from any state when a supplier cannot fulfil a booking (weather, capacity, force majeure). It is implemented as a XState v5 parallel state that runs alongside the main booking lifecycle.

This is the hardest part of the state machine to implement correctly. Mike should read the XState v5 parallel state documentation and the `ATP_KernelSpec_v1.docx` BOOKING_SUSPENDED section before writing this code.

---

## 5. Cedar — Why Policy-as-Code

Travel is a regulated industry. The same booking operation that is legal in Japan under OWN_SUPPLY may require a licensed travel agency under the Japan Travel Agency Act if it includes transport intermediation. The same activity supplier that is fully trusted in Nagano may need different compliance verification for a booking that crosses into another jurisdiction.

Encoding these rules in application code produces two problems. First, the rules are invisible — buried in conditionals that are hard to audit. Second, the rules are tightly coupled to the application — changing a policy requires a code deployment.

Cedar separates policy from application code. Cedar policy files are human-readable, auditable, and can be updated independently of the kernel. When the Japan Tourism Agency clarifies a regulatory requirement, the Cedar policy set is updated — not the kernel.

**Cedarling WASM in Node 24:**

The Cedarling WASM binding in Node 24 has known friction at integration time. If the WASM binding fails, `SEC-4` (the string-match fallback) is the safe path for Steps 1–4 of the build sequence. The fallback does not affect protocol correctness for early development — it does not block Steps 5–10. But Cedarling WASM must be working before the kernel is used in any production or staging environment.

---

## 6. The ATP Mandate Model

AI agents in the Activity Travel Protocol are not free-acting. They operate under mandates — explicit, cryptographically signed authorisation tokens that define exactly what an agent is permitted to do.

**Format:** Ed25519-signed JWT with media type `atp-mandate+jwt`

**Key fields:**
- `booking_object_id` — the mandate is bound to a specific Booking Object
- `authority_level` — 1 (query only) through 4 (full autonomy, human-equivalent)
- `permitted_transitions` — explicit list of state transitions the agent may request
- `delegation_ceiling` — the maximum authority level this agent may delegate to sub-agents

**The narrowing property:**

A delegated mandate cannot exceed the authority of the delegating mandate. If the Journey Coordinator holds authority Level 3, it cannot issue a Hotel Agent mandate at Level 4. The kernel enforces this at mandate issuance, not at transition time.

This is security-critical code. Mike reads this code personally. It does not get delegated to an AI agent or a junior contributor.

**The CONFIRMATION hard cap:**

No AI agent may confirm a booking autonomously above authority Level 1. Level 1 is "query only" — an agent at Level 1 can retrieve Booking Object state but cannot transition it to CONFIRMED. CONFIRMATION requires human approval (Level 2) or explicit high-authority mandate (Level 3+, which requires human issuance).

This is a protocol invariant. It cannot be configured away. It cannot be overridden by a Cedar policy. It is enforced in the Security Kernel mandate validation step.

---

## 7. The Event Log — Append-Only, Adapter Pattern

Every Booking Object state transition produces a `StateTransitionEvent` that is written to the event log before the transition completes. The event log is:

- **Append-only** — events are never updated or deleted
- **Adapter-based** — the kernel writes to an `IEventLog` interface, not directly to PostgreSQL
- **The source of truth** — Booking Object state can be reconstructed by replaying the event log

The adapter pattern means that swapping the event log implementation is a configuration change, not a code change. In development: `InMemoryEventLog`. In production: `PostgresEventLog`. In a hyperscale deployment: `KafkaEventLog`. The kernel code does not change.

The append-only constraint is deliberate. An audit trail that can be modified after the fact is not an audit trail. Regulators, insurance providers, and dispute resolution all depend on the immutability of the event log.

---

## 8. AI Agent Channels — WhatsApp and LINE

The choice of WhatsApp (international guests) and LINE (domestic Japanese guests and supplier notifications) is not a technical preference. It is a protocol decision based on channel penetration in the target market and the operational reality that:

- International guests arriving at Japanese ryokan and boutique hotels reliably have WhatsApp
- Japanese domestic travellers and all Japanese hospitality suppliers reliably have LINE
- Both channels support rich message formatting and webhook-based delivery confirmation

This decision is CLOSED. Do not propose alternative channels. Do not make channels configurable at the protocol level. Channel selection is declared in the Capability Declaration, not the kernel.

---

## 9. Backend First — Why Not Lovable First

The existing MyAuberge booking system (`booking.myauberge.jp`) is a React/Supabase application that writes directly to a database. There is no Booking Object. There is no state machine. There is no policy enforcement.

The temptation is to rebuild the frontend first — Lovable makes this fast and the result is immediately visible. This is the wrong sequence.

If the frontend is rebuilt before the ATP backend exists, the new frontend will either: (a) write directly to the database again, perpetuating the current architecture, or (b) call ATP endpoints that don't exist yet, requiring mocks that diverge from the real implementation.

The correct sequence: build the ATP backend (this repo) until the Azusa Journey E2E test passes end-to-end. At that point, the Lovable prompt is a clean, well-scoped job: "Build a booking UI that calls these ATP API endpoints." The frontend has no architectural decisions to make — they were all made by the backend.

---

## 10. The Azusa Journey — Why This Specific Test

The Azusa Journey was chosen as the acceptance test because it exercises every major protocol subsystem in a single realistic flow:

| Journey step | Protocol subsystem exercised |
|-------------|------------------------------|
| Guest books MyAuberge + Ponyhouse | Booking Object creation, multi-supplier trust chain |
| Payment captured | Pricing snapshot at CONFIRMATION, regulatory_class enforcement |
| AI agent mandate issued | Mandate model, narrowing property, authority level |
| Train departs Shinjuku | IN_JOURNEY phase entry, real-time event integration |
| Agent notifies guest at Shiojiri | WhatsApp delivery, Context Package, HEM readiness |
| Farm notified 30 min before arrival | LINE notification, supplier-facing agent scope |
| Guest arrives, checks in | IN_JOURNEY phase transition, participant records |
| Stay completes | Booking Object completion, event log finalisation |

A minimal booking test (hotel only, no activity, no AI agent) would leave most of the protocol untested. The Azusa Journey is deliberately maximal — if it passes, the kernel is conformant.

---

## 11. Three-Tier Scaling — Why NeMo Guardrails Are Absent at Tier 1

NeMo Guardrails (three normative rails: HEM Escalation, Notification Tone, Scope Boundary) are enforced at Tier 2 and Tier 3 but absent at Tier 1.

Tier 1 is development and small operator deployment — including MyAuberge's own production system in the early phase. NeMo at Tier 1 adds inference latency and cost that is disproportionate to the booking volume. The three rails are replaced at Tier 1 by prompt-layer instructions that achieve the same behavioural constraints without the inference overhead.

This is not a security compromise — the Security Kernel's mandate enforcement and Cedar policy evaluation run at all three tiers. NeMo is a behavioural layer on top of the policy enforcement layer, not a substitute for it.

---

## 12. Foundation / MyAuberge Separation — The Red Hat Model

The Activity Travel Protocol Foundation governs the protocol. MyAuberge K.K. is the Founding Member and commercial IaaS operator.

This separation is architecturally significant because it determines what belongs in this repo vs. what belongs to MyAuberge's commercial implementation:

| This repo (Foundation) | MyAuberge commercial implementation |
|----------------------|-------------------------------------|
| Protocol kernel (`@atp/core`, `@atp/security`, etc.) | `booking.myauberge.jp` webapp |
| Reference examples (Azusa Journey) | MyAuberge-specific Cedar policies |
| Conformance test suite | Ponyhouse Farm integration |
| OCTO Bridge (community-maintained) | IaaS managed services |
| OpenAPI specification | Nokia/SoftBank AI-RAN deployment |

The reference implementation in this repo is a **demonstration of protocol conformance**, not MyAuberge's production system. MyAuberge's production system builds on top of this kernel — it is not this kernel.

---

*Architecture document maintained by Tom Sato (ATP Foundation) and Mike Ma (MyAuberge K.K.).*  
*Spec: activitytravel.pro · GitHub: activity-travel-protocol*
