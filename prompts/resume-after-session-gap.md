# Prompt — Resume After Session Gap

Use this prompt at the start of any Claude Code session to re-establish full context. Claude Code has no memory between sessions — this prompt ensures nothing is missed.

---

## How to use

Copy the prompt below at the start of every new Claude Code session. Fill in the `[LAST SESSION]` and `[TODAY]` blocks. Takes 60–90 seconds but saves significant time diagnosing stale-context errors.

---

## Prompt

```
I am resuming work on the ATP Kernel — the reference implementation of the Activity Travel Protocol.

Read CLAUDE.md and ARCHITECTURE.md completely before proceeding. All decisions in those files are final.

Last session summary:
[LAST SESSION]
- Date: 
- Step(s) worked on: 
- What was completed: 
- What was left incomplete: 
- Any errors encountered that were not resolved: 
- Files created or modified: 
[/LAST SESSION]

Today's focus:
[TODAY]
- Step(s) to work on: 
- Specific task: 
- Any blockers I know about: 
[/TODAY]

Current repo state — run these commands and paste the output:
$ ls packages/core/src/
$ ls packages/security/src/
$ npm test 2>&1 | tail -30

Do not propose any changes until you have read CLAUDE.md, ARCHITECTURE.md, and the command output above. Start by summarising your understanding of where we are and what we are doing today.
```

---

## Why This Matters

Claude Code starts each session with no memory of previous work. Without this prompt:
- It may re-implement code that already exists
- It may propose architectural alternatives that were already considered and rejected (CLOSED decisions)
- It may not know which step of the build sequence is in progress
- It may not know about errors from the previous session that affect the current approach

The `npm test | tail -30` output is particularly valuable — a passing or failing test suite tells Claude Code the current state of the kernel immediately.

---

## Quick Version (for short sessions)

If you are continuing work within the same day and the context is fresh:

```
Continuing ATP Kernel work. Read CLAUDE.md.

Current step: [STEP NUMBER]
Current task: [ONE SENTENCE]
Last thing done: [ONE SENTENCE]

Here is the current output of the relevant file:
[PASTE FILE CONTENT]
```
