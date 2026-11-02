# One-Shot Orchestration

Per-repo config for the `one-shot` orchestrator and its cascade (`code-review`,
`code-fix`, `create-pr`, `create-commit`, `council-of-agents`).

## ADR writes (post-implement)

**Location:** `docs/adr/`

one-shot's step-7 ADR-write phase writes new ADRs here once code is implemented,
reviewed, and fixed — numbered after the highest existing `NNNN-<slug>.md`, one
commit per ADR (`docs(adr): <slug>`). ADRs document what actually shipped, not
what was planned.

ADRs are the **durable record**. The pipeline never deletes them. Reading ADRs
during the grilling phases follows `docs/agents/domain.md`; this is only the
**write** target. In multi-context repos, write context-scoped ADRs to
`src/<context>/docs/adr/`.

## Plans (single-PR lane)

**Location:** `plans/`

The single-PR lane reads its plan from `<location>/<feature>.md`. After ADRs are
committed (step 7b), one-shot retires the plan via a **committed deletion** —
plans are scratch, ADRs are durable. Multi-PR flows use PRDs/issues
(`docs/agents/prds.md`) instead, which `issue-worker` retires during implement.

## Auto mode

**Posture:** allowed _(set to "disabled" if `.claude/auto-mode-disabled` exists)_

- `auto` mode runs the cascade unattended (commits, pushes, opens a draft PR).
- If **`.claude/auto-mode-disabled`** exists at the repo root (or
  `~/.claude/auto-mode-disabled` globally), `auto` is vetoed — one-shot refuses
  `--mode=auto` and never silently downgrades to `hitl`/`grill`.
- Even when allowed, `auto` requires `--i-understand-the-risk` at invocation and
  is hard-capped: no `gh pr merge`, no force-push, no `--no-verify`, no writes to
  the default branch, no `gh issue create`.
- Drop the marker to opt this repo out entirely (recommended for shared / client
  repos). Delete it to opt back in.

## Base branch

**Base:** `<repo default>`

`code-review` diffs `git diff <base>...HEAD`; `create-pr` opens against `<base>`.
Auto-detected from the repo default branch if unset. one-shot refuses to run
`auto` mode when the current branch equals the base.
