# Prompt — Start Kernel Build Step

Use this prompt at the beginning of a Claude Code session to start or resume a specific kernel build step.

---

## How to use

Copy the prompt below into your Claude Code session, filling in the `[STEP]` and `[CONTEXT]` placeholders.

---

## Prompt

```
I am building the ATP Kernel — the reference implementation of the Activity Travel Protocol.

Read CLAUDE.md and ARCHITECTURE.md before proceeding. All architectural decisions in those files are CLOSED — do not propose alternatives.

I am working on **Step [STEP] of 10** of the kernel build sequence.

Step definitions (from CLAUDE.md):
- Step 1: PostgreSQL schema — Booking Object event log, Party Registry tables
- Step 2: Party Registry — in-memory store, Ed25519 key loading, regulatory_class
- Step 3: XState v5 state machine — 11 states, guard conditions, BOOKING_SUSPENDED interrupt
- Step 4: Security Kernel skeleton — 5-step execution path, mandate validation stub
- Step 5: Cedar policy evaluation — three policy sets (OWN_SUPPLY, FARM_EXPERIENCE, LICENSED_TA)
- Step 6: ATP Mandate Model — Ed25519-signed JWT (atp-mandate+jwt), narrowing property enforcement
- Step 7: Event log — append-only StateTransitionEvent, PostgresEventLog adapter
- Step 8: REST API — 17 OpenAPI 3.1 endpoints, ATPRuntime binding
- Step 9: Azusa Journey E2E test — full happy path, booking → in-journey → completed
- Step 10: BOOKING_SUSPENDED — cross-cutting interrupt, HEM escalation path

Additional context for this session:
[CONTEXT — describe what was done in previous sessions, any errors encountered, current file state]

Start by showing me what files exist in the relevant package directory, then propose the implementation plan for this step before writing any code.
```

---

## Example (Step 1)

```
I am building the ATP Kernel — the reference implementation of the Activity Travel Protocol.

Read CLAUDE.md and ARCHITECTURE.md before proceeding. All architectural decisions in those files are CLOSED — do not propose alternatives.

I am working on Step 1 of 10 of the kernel build sequence: PostgreSQL schema.

Additional context for this session:
This is the first session. The repo has CLAUDE.md, ARCHITECTURE.md, README.md, and package.json committed. No source code exists yet. Node.js 24 is installed. PostgreSQL 16 is running locally at localhost:5432, database name: atp_kernel_dev.

Start by showing me what files exist in the packages/core and packages/adapters directories, then propose the implementation plan for this step before writing any code.
```

---

## Notes

- Always give Claude Code the current file state — it has no memory between sessions
- Include any errors from the previous session verbatim — error messages are the fastest path to diagnosis
- "Propose before writing" is important for security-critical steps (4, 5, 6) — review the plan before code is generated
