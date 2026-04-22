# Contributing to the ATP Kernel

Thank you for your interest in contributing to the Activity Travel Protocol reference implementation.

The Activity Travel Protocol is governed by the [Activity Travel Protocol Foundation](https://activitytravel.org/governance) (一般社団法人アクティビティトラベルプロトコールファウンデーション), an independent open standards body constituted under Japanese law. The protocol specification and this reference implementation are published under the Apache 2.0 licence.

---

## Before You Start

### 1. Read the context documents

In order:

1. `README.md` — what this repo is
2. `CLAUDE.md` / `AGENTS.md` — architectural rules and closed decisions (read by AI coding tools automatically)
3. `ARCHITECTURE.md` — the reasoning behind every major design decision
4. [activitytravel.pro](https://activitytravel.pro) — the full protocol specification

Do not skip step 4. The protocol specification is the source of truth. This repo implements the spec — it does not define it.

### 2. Sign the CLA

All contributors must sign the Activity Travel Protocol Foundation Contributor Licence Agreement before a pull request can be merged.

**CLA:** [activitytravel.pro/working-drafts/governance/cla](https://activitytravel.pro/working-drafts/governance/cla/)

The CLA is modelled on the Apache Individual Contributor Licence Agreement. It assigns joint copyright to the Foundation while preserving your rights to use your own contributions.

If you are contributing on behalf of an organisation, contact [foundation@activitytravel.org](mailto:foundation@activitytravel.org) for the Corporate CLA.

---

## Development Setup

### Prerequisites

- Node.js 24+
- PostgreSQL 16+
- npm 10+

### Clone and install

```bash
git clone https://github.com/activity-travel-protocol/atp-kernel.git
cd atp-kernel
npm install
```

### Environment

Copy the example environment file and configure your local PostgreSQL connection:

```bash
cp .env.example .env
```

Required variables:

```
DATABASE_URL=postgresql://localhost:5432/atp_kernel_dev
NODE_ENV=development
ATP_LOG_LEVEL=debug
```

### Run tests

```bash
npm test                    # full test suite
npm run test:conformance    # Azusa Journey E2E conformance test
npm run test:unit           # unit tests only
```

### Start the kernel

```bash
npm run dev                 # development mode with hot reload
npm start                   # production mode
```

---

## Architecture Rules — Non-Negotiable

These rules are enforced at code review. PRs that violate them will not be merged regardless of other quality.

**The Security Kernel is non-bypassable.**  
Every Booking Object state transition must execute the full 5-step Security Kernel path. There is no skip flag, no test mode, no emergency bypass. Tests use the real Security Kernel against fixtures.

**One kernel, three configurations.**  
Do not fork the kernel for different operator types. Operator-specific behaviour lives in Cedar policy sets, selected by `regulatory_class`.

**The narrowing property is security-critical.**  
A delegated mandate cannot exceed the authority of the delegating mandate. This is enforced at mandate issuance in `@atp/security`. This code requires explicit review from a core maintainer before merge.

**The CONFIRMATION hard cap is a protocol invariant.**  
AI agents may not confirm bookings above authority Level 1. This cannot be configured away, overridden by Cedar policy, or relaxed for any environment.

**The event log is append-only.**  
`StateTransitionEvent` records are never updated or deleted. Do not add update or delete operations to the event log adapter.

**Do not reopen closed decisions.**  
See `ARCHITECTURE.md` and `CLAUDE.md` for the full list of closed decisions. If you believe a closed decision should be reconsidered, open a GitHub Discussion — do not submit a PR that assumes the decision has changed.

---

## Contribution Types

### Bug reports

Open a GitHub Issue using the **Bug Report** template. Include:
- Node.js version
- PostgreSQL version
- Minimal reproduction case
- The full `StateTransitionEvent` sequence leading to the bug, if relevant

### Protocol conformance issues

If you believe the implementation diverges from the protocol specification, open a GitHub Issue using the **Conformance Issue** template. Reference the specific specification section at activitytravel.pro.

Note: conformance issues may require a specification change (raised at [activity-travel-protocol/protocol-spec](https://github.com/activity-travel-protocol/protocol-spec)) rather than a kernel change. The spec is the source of truth.

### Feature contributions

For new features, open a GitHub Discussion before writing code. Features that affect the Security Kernel, the mandate model, or the Booking Object state machine require Foundation TSC review before implementation begins.

Features that add new `@atp/` packages or extend existing packages without modifying core protocol behaviour can proceed to PR directly after discussion.

### Documentation

Documentation PRs (README improvements, JSDoc additions, example updates) are welcome without prior discussion.

---

## Pull Request Process

1. Fork the repository and create a branch from `main`
2. Branch naming: `feat/short-description`, `fix/short-description`, `docs/short-description`
3. Write or update tests for any changed behaviour
4. Ensure the full test suite passes: `npm test`
5. Ensure the Azusa Journey conformance test passes: `npm run test:conformance`
6. Update relevant JSDoc if you change a public API
7. Open a PR against `main` with a clear description referencing any related Issues

### PR description template

```
## What this changes
[One paragraph describing the change]

## Why
[Reference to Issue, Discussion, or specification section]

## Protocol invariants verified
- [ ] Security Kernel non-bypassable
- [ ] Narrowing property enforced
- [ ] CONFIRMATION hard cap intact
- [ ] Event log append-only

## Tests added/updated
[List of test files changed]
```

### Review

PRs require approval from at least one core maintainer. PRs that touch `@atp/security`, the mandate model, or the Booking Object state machine require approval from two core maintainers.

---

## Commit Style

We use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat(@atp/core): add BOOKING_SUSPENDED interrupt to state machine
fix(@atp/security): enforce narrowing property at mandate issuance
docs(examples): add Ponyhouse supplier example
test(conformance): extend Azusa Journey to cover HEM escalation path
```

Scope is the affected package name where possible.

---

## Governance

The Activity Travel Protocol Foundation Technical Steering Committee (TSC) is responsible for protocol specification decisions. Kernel implementation decisions that do not affect protocol semantics are made by the core maintainer team.

For Foundation membership enquiries: [activitytravel.org](https://activitytravel.org)  
For TSC processes: [activitytravel.pro/working-drafts/governance](https://activitytravel.pro/working-drafts/governance/)  
For general questions: open a GitHub Discussion

---

*Activity Travel Protocol Foundation · Apache 2.0 · activitytravel.org*
