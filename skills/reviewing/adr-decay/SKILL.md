---
name: adr-decay
description: Report-only decay sweep of a repo's ADR corpus against the current code, other ADRs, and the issue tracker. Classifies every finding against a fixed taxonomy — decayed / born-wrong / not-functioning-as-a-convention — with file:line evidence, then routes each to a factual amendment or a fresh decision. Never edits an ADR. Use when ADRs have gone stale, when cleaning up or auditing the ADR folder, before trusting an old ADR, or when an ADR contradicts the code.
---

# ADR Decay

Sweep an ADR corpus for the ways it stops being trustworthy. The output is a **findings report plus proposed amendments**, not edited ADRs. Never commit, never auto-edit, never delete.

Work the whole taxonomy. Dead references are the cheapest class and the least harmful; an ADR whose Decision section no longer states an actionable decision passes every link check ever written.

## The taxonomy

### Axis A — Decayed: was true, no longer

| # | Class | Tell |
|---|---|---|
| A1 | **Dead pointer** | Cites an artifact outside the repo — a branch, a deleted PRD, a vendor console, a Notion page, a sha, a line number. |
| A2 | **Code drift** | Asserts a fact about the codebase that has since changed ("`X.status` is a free-text string"). |
| A3 | **Unmarked contradiction** | A later ADR overrode this one and neither is annotated. |
| A4 | **Status lies** | Header says `Accepted`; in fact superseded in whole or in part. |
| A5 | **Incoherent after amendment** | N dated updates have accreted; you must diff the layers to learn what's true. |
| A6 | **Decided, never built** | Accepted, nothing implements it, nothing records the abandonment. |
| A7 | **Zombie open question** | Listed as deferred; resolved elsewhere and never struck. |

### Axis B — Born wrong: never was true

| # | Class | Tell |
|---|---|---|
| B1 | **False premise** | A claim about the code, a vendor, or the platform that was never checked. Git history shows no change — verify the claim, not the diff. |

### Axis C — True, but not doing its job

| # | Class | Tell |
|---|---|---|
| C1 | **Fix list, not a convention** | Says what changed rather than how to build this from now on. |
| C2 | **Unenforced rule** | States a convention with no check anywhere. |
| C3 | **No bypass** | A rule with no sanctioned escape hatch and no way to leave a reason. |
| C4 | **Rotting question** | A question the ADR raised, still unanswered while the thing it guards hardens. Or a client/vendor question parked here instead of the project's open-questions home. |
| C5 | **Missing ADR** | A decision made and never recorded, often because it lived in a doc deleted after delivery. Invisible from inside the corpus — only Phase 4 finds it. |

## Phase 0: Locate and inventory

Find the ADR directory (`docs/adr/`, `**/docs/adr/`, `doc/architecture/decisions/`). Record per ADR: number, title, date, status line. Measure the corpus (`wc -l`) — the size decides Phase 3's harness.

Also locate: the issue tracker and how to query it, the living docs (`CONTEXT.md`, `README`), the designated home for unanswered questions, and any stated ADR-retention policy. Report what you found.

## Phase 1: Mechanical pre-pass

Scripts, not models. Settle here everything a script can settle:

- **A1** — extract every reference-shaped token (relative paths, doc paths, branch names, `git show`, shas, URLs, issue refs, `file:line` citations) and resolve each: `test -e`, `git rev-parse --verify`, `git cat-file -e`, `gh issue view`.
- **A4** — grep the corpus for "supersed\*" / "amend\*" mentioning ADR N, then diff that set against what N's own status line admits.

## Phase 2: Cluster by topic

**Never partition per-file.** A3, A7 and C5 are only visible with several ADRs in one context. Group by subject matter — per bounded context or subsystem — each cluster holding every ADR that touches it. An ADR may appear in two clusters.

Report the clustering before proceeding; a bad partition caps what the sweep can find.

## Phase 3: Read each cluster against the taxonomy

Work all classes except C5. Each finding carries:

- the class (A1–C4),
- `file:line` **and the quoted line**,
- for A2/B1, the counter-evidence in the code (`file:line`, symbol),
- one sentence on what is true now.

No evidence → no finding. "Probably stale" is not a finding.

**Harness.** Under ~4 clusters, or a corpus that fits comfortably in context, read directly. Beyond that, one subagent per cluster — never one per ADR. Each returns evidence-bearing findings only, not prose summaries. For large sweeps, have each cluster write findings to a scratch file.

## Phase 4: Reverse pass — decisions with no ADR (C5)

Run from the code toward the corpus. Look at enforced conventions, CI checks, lint rules, architectural seams, and recently merged work; for each, ask which ADR explains *why*. Anything load-bearing with no ADR behind it is a C5 finding.

Look hardest where the project deletes specs after delivery — if the rationale lived there and the harvest into an ADR was partial, only this pass notices.

## Phase 5: Verification gate

Re-check every finding yourself against the tree before it enters the report, especially anything a subagent returned. Doc-claim findings are wrong at a high rate in both directions — reporting zero instances where there are several, and asserting an artifact is missing without grepping.

An unverified finding is **dropped**, not downgraded.

## Phase 6: Report and route

No edits applied.

### Inventory
Phase 0 findings plus the Phase 2 clustering.

### Findings
One table: ADR, class, `file:line`, quoted line, what's true now. Ordered by damage — an A4/A5 that misleads every reader outranks an A1 that dead-ends one.

### Routing
Each finding goes to exactly one of:

- **Factual amendment** — the world-fact changed, the decision stands. Just an edit; don't convene a design session for a correction.
- **Decision change** — the decision itself no longer holds. Needs a real design conversation and a new ADR superseding the old section. Say so; never smuggle a decision into a cleanup PR.
- **Question** — route to the project's open-questions home, not a fresh ADR paragraph.
- **New ADR** (C5) — name the decision to record and where its rationale currently lives, if anywhere.

### Proposed amendments
Text to apply by hand, in the corpus's existing amendment style. Strike and annotate; never launder:

```
~~original claim~~ **Update (YYYY-MM-DD): <what's true now>** — see ADR-N.
```

For A5, the amendment is usually restructuring: hoist the current decision to the top of the Decision section and demote accreted updates to history.

State plainly: "apply these yourself; I have not edited any ADR."

### Verdict
Per ADR: *Sound* / *Amend (N findings)* / *Decision needs revisiting* / *Superseded in fact*. Corpus-level: how many C5 holes.

## Rules

- **Report-only.** Never edit, commit, or delete an ADR. Check the project's retention policy; default to never delete — superseded ADRs are kept for the reversal narrative.
- **Evidence or it's dropped.** `file:line` plus the quoted line.
- **A correction is not a decision.** Route to a fresh design conversation only when the *decision* changes.
- **Cluster, never per-file.**
- **Don't manufacture findings.** An entirely sound ADR is a real result; say so.
- When the user applies amendments, land them in a worktree, not the main checkout, as one docs-only PR.
