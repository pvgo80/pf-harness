---
name: prd-drift
description: Report-only drift check of a PRD against the current code, issues, ADRs, and living docs. Partitions the PRD's claims into machine-checkable vs. human-judgment, verifies only the checkable ones with concrete evidence, and proposes annotations — never rewrites the PRD. Use when a PRD may have gone stale, before retiring/archiving a PRD, or to decide whether a learning-phase PRD's goals are met.
---

# PRD Drift

Reconcile a PRD against what the codebase actually became. The output is a **drift report plus proposed edits**, not a rewritten PRD. This skill never commits, never auto-edits, and never deletes — it tells you where the spec and reality have diverged and hands you the annotations to apply.

The governing distinction: a PRD makes two kinds of claims, and only one is machine-checkable.

- **Checkable claims** — falsifiable against code, issues, or ADRs. "1:1 mirror of the REST surface," "writes gated by a permission prompt," "X is a non-goal," "audit log captures per-session sequences." These you verify.
- **Judgment claims** — outcomes and aspirations measurable only from running-system evidence and human decision. "≥3 sessions/week," "champion actively using it," "≥2 distinct flows emerged." These you **never** rule on. You surface them and point at where the evidence lives.

Conflating the two is the classic failure mode: an unconstrained pass will confidently mislabel aspirational goals as bugs. The partition in Phase 1 is the whole point of this skill.

## Phase 0: Locate inputs

Find the four sources. Adapt to the project; don't assume a layout.

1. **PRD location.** Check `docs/agents/prds.md` for a `**Location:**` line and use that path; else default to `prds/` (and `packages/*/prds/` in a monorepo). If several PRDs exist, ask which one to check.
2. **Issues.** Resolve the issues mode the way `/prd-to-issues` does: the `**Issues:**` line in `docs/agents/prds.md` (`tracker` or `file`); absent that, `tracker` when the file says in prose that issues live on the tracker or `docs/agents/issue-tracker.md` names GitHub or GitLab, else `file`. `tracker`: the PRD's issues carry the `prd:<feature>` label. `file`: a local `<feature>-issues.md` next to the PRD. Record how to list open vs. closed (`gh issue list --label <x> --state open`, or the file's unchecked boxes).
3. **ADRs.** Look for `docs/adr/` or `**/docs/adr/`. ADRs are the authoritative record of decisions that may have *superseded* the PRD.
4. **Living docs.** `CONTEXT.md`, `CONTEXT-MAP.md`, `README`, architecture notes — the "current truth" the PRD should be consistent with.

Report what you found and where. If a source is missing, note it and proceed with the rest.

## Phase 1: Partition the claims

**Do not verify anything yet.** Read the PRD top to bottom and extract every concrete claim — Goals, Non-goals, scope bullets, architecture statements, success criteria. For each, tag it:

- `[CHECK]` — falsifiable against code / issues / ADRs.
- `[JUDGE]` — an outcome or aspiration that needs running-system evidence or a human call.

Present this partitioned list and let the user correct the tagging before you spend effort verifying. Mis-tagging here poisons the whole report.

Already-annotated claims (e.g. a struck-through `~~non-goal~~ **Reversed** — see ADR-N`) are *not* drift — they're resolved history. Note them as "already reconciled" and skip.

## Phase 2: Verify the `[CHECK]` claims

For each checkable claim, assign one status, **with evidence** (`file:line`, issue #, ADR #). No evidence → you can't claim it; mark `Unverifiable` and move on.

- **Holds** — code/issues confirm the claim. Cite where.
- **Drifted** — reality differs from the claim (scope narrowed, behavior changed, mechanism replaced). Cite both what the PRD says and what the code does.
- **Reversed** — a non-goal got built, or a goal got explicitly dropped, and an ADR or closed issue records the decision. Cite the ADR/issue.
- **Unbuilt** — claimed as in-scope, no implementing code, issue still open. Cite the open issue.

Trace to real code as in a branch review — point at the symbol/handler/schema, don't argue "it probably." If you can't find evidence in a couple of minutes of reading, it's `Unverifiable`, not `Drifted`.

## Phase 3: Reverse-drift sweep

Drift runs both ways. Verifying the PRD's own bullets only catches what the PRD *thought to mention*. Now look for what the code, ADRs, and closed issues contain that the **PRD never anticipated**:

- New scope shipped that the PRD doesn't describe.
- ADRs dated after the PRD that change its premises.
- Decisions made in issues/PRs that contradict a PRD statement.

These are drift too — the PRD is stale by omission. List them with evidence.

## Phase 4: Report

Five sections. No edits applied — this is a report.

### Inputs
What you found in Phase 0 (PRD path, issue source, ADR dir, living docs).

### Drift table
Every `[CHECK]` claim with its status (Holds / Drifted / Reversed / Unbuilt / Unverifiable) and one-line evidence. Plus the Phase-3 reverse-drift items.

### Proposed edits
Concrete annotations to apply **by hand** — preserve history, don't launder it. Prefer the strikethrough convention:
- `Drifted`/`Reversed` → propose `~~original claim~~ **Reversed/Drifted** — <what's true now>, see ADR-N`.
- `Unbuilt` → propose a status note, not deletion.
- A new ADR where a decision was made but never recorded.

Present these as a suggested diff. State plainly: "apply these yourself; I have not edited the PRD."

### Judgment claims (escalated)
List every `[JUDGE]` claim verbatim. Do **not** rule on them. For each, name where the evidence would come from (audit log, analytics, a project introspection skill, user self-report). This is the human's call.

### Verdict
Distinguish the two senses of "complete" explicitly:
- **Implementation:** *Matches* / *Drifted (N stale claims)* / *Incomplete (N unbuilt)* — derived from `[CHECK]` + issue state. This you can assert.
- **Goals:** *Cannot assert — see escalated judgment claims.* Recommend the harvest-then-retire path if implementation is done: extract the still-true decisions into ADRs / living docs, then archive the PRD. Never recommend deleting a PRD whose system is still in production.

## Rules

- **Report-only.** Never edit, commit, archive, or delete. Propose; the user applies.
- **Never rule on `[JUDGE]` claims.** Surface and route them. An LLM "verifying" an outcome goal is the thing this skill exists to prevent.
- **Evidence or it's `Unverifiable`.** No `file:line`/issue/ADR, no status.
- **Preserve history.** Propose strikethrough + annotation, not silent rewrites — the reversal narrative is part of the PRD's value.
- A PRD that fully Holds is a valid, useful result. Say so plainly; don't manufacture drift to look thorough.
