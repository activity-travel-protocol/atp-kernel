# Prompt — Add a New Booking Object State Transition

Use this prompt when you need to add a new state transition to the Booking Object state machine. This is an advanced operation that touches the Security Kernel — review the generated code carefully before committing.

---

## How to use

Copy the prompt below into your Claude Code session, filling in the transition details.

---

## Prompt

```
I need to add a new state transition to the ATP Booking Object state machine.

Read CLAUDE.md and ARCHITECTURE.md before proceeding. This change touches the Security Kernel — review the proposed changes with me before writing any code.

The new transition details are:

Transition name: [e.g. ACTIVITY_ISSUE_REPORTED]
From state(s): [e.g. IN_JOURNEY:ACTIVITY]
To state: [e.g. IN_JOURNEY:ACTIVITY — same state, or a different state]
Trigger: [what causes this transition — human action, agent action, external event]
Authority level required: [1 | 2 | 3 | 4]
Regulatory classes where permitted: [OWN_SUPPLY | FARM_EXPERIENCE | LICENSED_TA | ALL]
HEM trigger: [YES — which HEM entry | NO]
Description: [what this transition represents in the booking lifecycle]

Spec reference (if any): [activitytravel.pro URL or spec document section]

Before writing code, show me:
1. The current state machine definition in packages/core/src/state-machine/
2. The relevant Cedar policy sets in packages/security/src/cedar/
3. The StateTransitionEvent types in packages/schema/src/

Then propose the changes required across all affected files. I will review and approve before you write any code.

The changes will affect:
- packages/core/src/state-machine/ — XState machine definition
- packages/security/src/cedar/ — Cedar policy sets (for each applicable regulatory_class)
- packages/schema/src/ — transition type additions
- packages/workflow/src/ — if HEM trigger applies
- tests/conformance/ — conformance test coverage
- fixtures/ — update affected Booking Object snapshots
```

---

## Review Checklist

Before approving the generated code, verify:

- [ ] The new transition is defined from the correct `from` states only — no unintended paths
- [ ] The Cedar policy addition uses `ALLOW` explicitly — Cedar denies by default
- [ ] The authority level is appropriate — CONFIRM-equivalent transitions must not be at Level 1
- [ ] If HEM trigger: the HEM entry number is correct and the escalation path is defined
- [ ] The conformance test covers the new transition in both the happy path and rejection case
- [ ] The spec reference is correct — do not add transitions not in the protocol specification

---

## Notes

- New transitions must be in the protocol specification before they are implemented — check activitytravel.pro first
- If the transition you need is not in the spec, the correct path is a protocol change proposal at activity-travel-protocol/protocol-spec, not a kernel change
- BOOKING_SUSPENDED is a parallel state — transitions that interact with it require extra care; bring this to Tom Sato before proceeding
