# One-Shot

Skills for taking a feature end-to-end — plan → implement → review → fix → ship —
with a self-contained review cascade that runs on the local diff before anything
is pushed.

Pairs with the `building` plugin: `building` supplies the grilling and
prototyping foundation, `one-shot` supplies the orchestrator, its PRD lane
(write-a-prd → prd-to-issues → issue-worker/issue-swarm), and the
review/fix/PR machinery.

## Skills

- **[one-shot](./one-shot/SKILL.md)** — Orchestrate a feature end-to-end (plan → implement → verify → review → fix → simplify → verify → ADR write → ship) with three autonomy modes (hitl / grill / auto, default grill) and parallel fan-out per phase.
- **[aoe-orchestrate-plan](./aoe-orchestrate-plan/SKILL.md)** — Work a group of GitHub issues sharing a label as stacked PRs: a resident orchestrator session spawns one aoe worker session + worktree per issue, running independent lanes of the dependency graph in parallel (restack workers give multi-parent issues a single base), each commit passes an adversarial subagent review, workers report completion packets back, and the close-out synthesizes them, plans follow-up issues, and merges + cleans up on the human's yes. Requires the external `aoe` session manager (validated prerequisite — halts without it). Trigger: "orchestrate the plan" / "run the stacked pipeline on <tag>" — distinct from one-shot's "run the pipeline".
- **[aoe-lite](./aoe-lite/SKILL.md)** — Lightweight aoe orchestration for projects that have not adopted pf: one private playbook at `~/.pf-aoe/<KEY>/playbook.md` holds how the project operates and your preferences, nothing is written to the repo, and workers can be any agent. `/aoe-lite setup`, then `/aoe-lite <what to work on>`.
- **[council-of-agents](./council-of-agents/SKILL.md)** — Parallel multi-lens planning amplifier: 3-5 subagents (architecture, UX, security, ops, persistence, etc.) synthesised into a unified design brief.
- **[code-review](./code-review/SKILL.md)** — Parallel lens-based review on a local diff (default) or an open PR: one agent per lens (security / correctness / plan-vs-diff / perf / nit / CI), merged into one must-fix / should-fix / consider / nit findings packet. Local draft by default; `--post` to GitHub when a PR exists.
- **[code-fix](./code-fix/SKILL.md)** — Consume a findings packet, fan out one fix worker per file-partition (each in its own worktree), cherry-pick all commits onto the head branch. Local mode stops at commits; PR mode pushes and replies on threads.
- **[create-pr](./create-pr/SKILL.md)** — Open a PR from the current branch with a Conventional Commits title and a WWH body (What / Why / How). Draft by default.
- **[create-commit](./create-commit/SKILL.md)** — Stage and commit with Conventional Commits. Never adds Claude as co-author; HITL gate by default.
- **[write-a-prd](./write-a-prd/SKILL.md)** — Create a product requirements document through user interview, codebase exploration, and module design.
- **[prd-to-issues](./prd-to-issues/SKILL.md)** — Break a PRD into independently-grabbable issues using tracer-bullet vertical slices.
- **[issue-worker](./issue-worker/SKILL.md)** — Pick up the next uncompleted issue, implement it using TDD, and commit. One invocation = one issue = one commit.
- **[issue-swarm](./issue-swarm/SKILL.md)** — Fan-out up to N issues in parallel, each in its own worktree, gated by a reviewer subagent before being marked done.
- **[setup-pf-skills](../setup/setup-pf-skills/SKILL.md)** (in the `setup` plugin) — Scaffold the per-repo `## Agent skills` block + `docs/agents/` so the orchestrator knows domain-doc layout, PRD/issue/plan locations, post-implement ADR write target, and auto-mode posture. Run once per repo.

## External soft dependencies

Not in this marketplace; the pipeline degrades gracefully without them.

| Skill | Used for | Without it |
| ----- | -------- | ---------- |
| `wayfinder` | Multi-session planning front-end for foggy or oversized briefs (step 1b) | Falls back to the PRD lane |
| `tdd` | The implement phase's red → green → refactor contract | Any equivalent TDD skill works |
| `diagnosing-bugs` | Diagnosis loop when implement or review hits a hard bug | Ad-hoc debugging |
| `code-writing-standards` | House charter the correctness lens enforces | Falls back to `CODING_STANDARDS.md` / `CONTRIBUTING.md`, then the Fowler baseline |
| `superpowers` | `writing-plans` + `subagent-driven-development` | Any planning skill that emits a plan-file path |

`aoe-orchestrate-plan` and `aoe-lite` additionally depend on the **`aoe` session manager
CLI** — a hard prerequisite, not a skill: it validates `command -v aoe` up
front and halts legibly when it's absent. `aoe-orchestrate-plan`'s constants (`WORKER_CMD`,
`MERGE_CMD`) are repo-overridable via `docs/agents/aoe-orchestrate-plan.md`.

`verify` and `simplify` are built-in Claude Code skills — invoked via the Skill
tool, not vendored here. On an older Claude Code release where either is
missing, `one-shot` steps 3b and 6b carry the fallback contract in full (drive
the surface and capture evidence; four-lens behavior-preserving cleanup), so
the gate is run by hand rather than skipped.
