---
name: review-branch
description: Multi-phase critical review of the current branch against main. Loads system context before reading the diff, then validates every finding against concrete execution paths. Use for a deep, human-facing pre-merge review of a whole branch — when you want proven-reachable findings rather than a machine findings packet feeding an automated fix cascade.
---

# Branch Review

A structured, multi-phase review of the current branch. The goal is accuracy over volume — only flag issues you can prove are reachable.

## Phase 1: Scope & System Context

**Do NOT read the diff yet.**

1. Run `git diff main...HEAD --stat` to see which files changed.
2. For each changed file, read its **callers and consumers** — not the file itself. Understand:
   - Who calls the changed functions/methods?
   - What are the entry points? (HTTP handlers, job workers, CLI commands, event listeners, etc.)
   - What are the state machines, lifecycles, or domain invariants the surrounding system relies on?
   - What external systems or side effects are involved?
3. If a CLAUDE.md, README, or architecture doc exists for the affected modules, read it.
4. Summarize your understanding as **"System Context"** so the user can correct misunderstandings before you proceed.

## Phase 2: Diff Analysis

Now read the full diff with `git diff main...HEAD`.

1. State the **intent** of the change in one sentence — what problem is it solving?
2. Map each modification to the system context from Phase 1.
3. List candidate concerns — things that _look_ like they could be issues. Do NOT present these to the user yet. These are hypotheses, not findings.

## Phase 3: Validate Findings

For each candidate concern from Phase 2:

1. **Trace a concrete execution path.** Name the specific caller, the sequence of events, and the preconditions required. Not "what if X happens" — show _how_ X happens by pointing at real code.
2. **Check if the system permits it.** Apply the relevant validation from the Appendix below (concurrency, data integrity, API contracts, etc.). If the system's design prevents the scenario, drop it.
3. **Distinguish theoretical from reachable.** If you can't construct a step-by-step path through actual code that leads to the problem, it's theoretical. Drop it.
4. **If you can't trace a path in under 2 minutes of reading, it's probably not real.** Move on.

Only concerns that survive this phase become findings.

## Phase 4: Assessment

Present the review in three sections:

### System Context
The summary from Phase 1 (2-4 sentences).

### Intent
What the change does and why (1-2 sentences from Phase 2).

### Findings
Each validated finding with:
- **Severity**: high / medium / low
- **Concrete path**: the traced execution path that makes it reachable
- **Suggestion**: what to do about it

If there are zero findings, say so. An empty findings section is a valid review outcome — it means the change is sound.

### Verdict
One of:
- **Ship it** — no issues, or only cosmetic ones
- **Ship with nits** — minor improvements possible but not blocking
- **Needs changes** — at least one high/medium finding that should be addressed

## Rules

- Never pad the review with theoretical concerns to look thorough.
- Never raise an issue without naming the specific code path that makes it reachable.
- If the change is good, say it's good. A clean review is more useful than a noisy one.

---

## Appendix: Domain-Specific Validations

When a candidate concern touches one of these domains, apply the corresponding checks before promoting it to a finding.

### Concurrency & Race Conditions
- Identify the concurrent actors (workers, handlers, cron jobs).
- Determine serialization guarantees (job queue `teamSize`, DB locks, mutex, single-threaded event loop).
- Prove two actors can actually overlap on the same data. If the queue serializes them, or they operate on disjoint status partitions, drop the concern.

### State Machine Integrity
- Map the valid transitions (e.g., pending -> initiated -> confirmed).
- For a "status regression" concern, trace which code path could move state backwards. Name the caller and the conditions.
- Check whether guards (WHERE clauses, precondition checks) prevent the invalid transition.

### Data Integrity & Atomicity
- If the concern is about partial writes, check whether the operations are wrapped in a DB transaction or are a single atomic statement.
- If they're separate statements, identify what happens on crash between them. Is the resulting state recoverable by an existing mechanism (retry, monitor, etc.)?

### API Contracts & External Systems
- If the concern involves an external API (custodial provider, payment gateway, etc.), check the API's idempotency and retry guarantees.
- Verify that the code handles the external system's error/success states correctly. Check what happens on timeout vs rejection vs success.

### Authorization & Input Validation
- If the change touches an endpoint or handler, verify that auth checks are in place.
- For user-facing inputs, check for validation at system boundaries (Zod schemas, guards, etc.).
