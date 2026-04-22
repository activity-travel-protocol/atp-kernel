# Prompt — Debug a Security Kernel Rejection

Use this prompt when a Booking Object state transition is being rejected by the Security Kernel and you need to diagnose why.

---

## How to use

Copy the prompt below into your Claude Code session, filling in the error details. Include the full error output — do not summarise it.

---

## Prompt

```
A Booking Object state transition is being rejected by the ATP Security Kernel. I need to diagnose the rejection.

Read CLAUDE.md and ARCHITECTURE.md before proceeding. Pay particular attention to the Security Kernel 5-step execution path in ARCHITECTURE.md Section 3.

The rejection details are:

Booking Object ID: [bo_xxx]
Requested transition: [e.g. CONFIRM, TRANSITION_PHASE, CANCEL]
Current Booking Object state: [e.g. PENDING_CONFIRMATION]
Mandate presented:
  - mandate_id: 
  - authority_level: 
  - permitted_transitions: []
  - issued_to: 
Operator regulatory_class: [OWN_SUPPLY | FARM_EXPERIENCE | LICENSED_TA]

Full error output:
[PASTE FULL ERROR INCLUDING STACK TRACE]

Full Security Kernel log output (if available):
[PASTE KERNEL LOG]

Diagnose which of the 5 Security Kernel steps failed:
1. Mandate validation (signature, authority level, booking_object_id binding, permitted_transitions list)
2. Cedar evaluation (policy set mismatch, transition not permitted for this regulatory_class)
3. XState transition (guard condition failed, illegal state transition)
4. Event log write (database error, constraint violation)
5. Event emission (subscriber error — rare)

Show me the relevant source code in packages/security/src/ and explain what is failing and why. Then propose the fix.

Do NOT suggest bypassing the Security Kernel. Do NOT suggest adding a skip flag or test mode. The fix must be within the normal execution path.
```

---

## Common Causes by Step

**Step 1 — Mandate validation failures:**
- `MANDATE_INVALID`: Ed25519 signature verification failed — check key rotation, clock skew
- `BOOKING_OBJECT_MISMATCH`: mandate's `booking_object_id` doesn't match the transition target
- `AUTHORITY_INSUFFICIENT`: requested transition requires higher authority level
- `TRANSITION_NOT_PERMITTED`: transition not in mandate's `permitted_transitions` list
- `CONFIRMATION_CAP`: CONFIRM requested by autonomous agent — protocol hard cap, not a bug

**Step 2 — Cedar evaluation failures:**
- `CEDAR_DENY`: policy explicitly denies this transition for this `regulatory_class`
- `POLICY_NOT_FOUND`: no policy set loaded for this `regulatory_class` — startup configuration issue
- `CEDARLING_ERROR`: WASM binding error — check Node 24 compatibility, consider SEC-4 fallback

**Step 3 — XState failures:**
- `ILLEGAL_TRANSITION`: transition not defined from current state — check state machine definition
- `GUARD_FAILED`: XState guard condition returned false — check guard implementation

**Step 4 — Event log failures:**
- `UNIQUE_CONSTRAINT`: duplicate event ID — check UUID generation
- `CONNECTION_ERROR`: PostgreSQL connection lost — check DATABASE_URL and connection pool

---

## Notes

- Always include the full error output — summarised errors lose the information needed for diagnosis
- Check the event log (`packages/adapters/src/postgres-event-log.ts`) for the sequence of events leading to the rejection — the Booking Object state at the time of rejection matters
- The Security Kernel log should show which step failed — enable `ATP_LOG_LEVEL=debug` if not already set
