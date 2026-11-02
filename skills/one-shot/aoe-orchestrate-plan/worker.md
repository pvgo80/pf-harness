# Worker protocol (`--worker`)

You are one spoke, running in the main checkout root. Your jurisdiction is
exactly: this issue, your worktree, your PR, your packet. You do not select
issues, spawn sessions, or merge PRs — ever. Establish coordinates:

- ISSUE_N: from `--issue`. BASE_BRANCH: from `--base`. STACK_BASE: from
  `--stack-base`. ORCH_TITLE: from `--orchestrator` — your reply address.
- MY_BRANCH: per the Constants branch naming. Without `--resume` it must not
  exist yet — if it does, something is mis-wired: **fail** (below), don't
  guess.
- MY_TITLE: `aoe session current`.

With `--notes`, read the file before step 0: open concerns from the issues
this one builds on. Handle what falls in this issue's scope; list the rest
under `Deferred` in your packet.

## Context discipline (not optional)

Delegate everything that is not a design decision to subagents with fresh
context: exploration, greps, environment setup and its logs, running gates,
screenshots. Keep only decisions and subagent summaries in your main
context. At ~60% context, stop at a phase boundary, commit, run
`/building:handoff` into the worktree root, and follow **Handoff** below.

## 0. Create your worktree

(Skip on `--resume`: cd into the existing worktree, read the handoff
document, and continue from where it left off.)

From the root, the repo's own convention wins: if `docs/agents/worktrees.md`
exists, follow its flow with MY_BRANCH off BASE_BRANCH. Fallback:

```bash
git worktree add ../<repo-name>-<FEATURE>-issue-<ISSUE_N> -b MY_BRANCH BASE_BRANCH
```

Then hydrate whatever the bootstrap doesn't cover: untracked secret env files
(`.envrc.private` and kin) and gitignored resources the issue references —
copy from the main checkout, or point at its copy. Do this before writing
code, not after the app mysteriously fails.

All build/test/git work happens **inside the worktree** (cd into it per
command); the session itself stays rooted in the main checkout. Other
workers run beside you in their own worktrees — use the ports, databases,
and containers `docs/agents/worktrees.md` assigns to yours.

## 1. Work the issue — with its adversarial subagent review

Read the issue with `gh issue view <ISSUE_N>`, then resolve the repo's
issues mode the way `/issue-worker` does (the `**Issues:**` line in
`docs/agents/prds.md`; absent that, `tracker` when the file says in prose that
issues live on the tracker or `docs/agents/issue-tracker.md` names GitHub):

- **`tracker` mode** → invoke `/issue-worker`, constrained: work exactly
  issue **ISSUE_N** (no self-selection), in the worktree from step 0 on
  MY_BRANCH, do **not** open a PR (step 2 opens it stacked), and leave the
  PRD in place even on the last issue (the synthesis reads it). Its
  **adversarial pre-commit subagent review stays ON**.
- **`file` mode** → do not invoke `/issue-worker` (it
  would look for a local `*-issues.md` file and delete PRD artefacts this
  pipeline needs). Follow its discipline inline:
  - TDD: failing tests first, then implement, then verify green.
  - Only what the issue asks for — no scope creep.
  - Read neighbouring code first; follow existing patterns.
  - **Before committing, dispatch an adversarial reviewer subagent** exactly
    per issue-worker's review block: fresh agent, diff + issue + test
    command, default stance "broken until proven otherwise",
    fix-and-re-review up to 3 cycles, stop and report if still failing.
  - If blocked or unsure, do NOT guess — **fail** (below).

## 2. Open the stacked PR

From inside the worktree:

```bash
git push -u origin MY_BRANCH
gh pr create --base BASE_BRANCH \
  --title "<issue title>" \
  --body "<summary>

Closes #<ISSUE_N>"
aoe session set-base "$(aoe session current -q)" BASE_BRANCH
```

Capture PR_N from the output. `--base BASE_BRANCH` keeps the PR diff to this
slice only; `Closes #<ISSUE_N>` closes the issue on merge; `set-base` makes
the aoe diff view match the PR.

## 3. Report completion — your last act

Write `<MSG_DIR>/<FEATURE>-issue-<ISSUE_N>-packet.md`:

```
WORKER COMPLETE #<ISSUE_N>
PR: #<PR_N> (MY_BRANCH, base BASE_BRANCH)
Shipped: <one paragraph — what exists now that didn't>
Decisions: <choices the issue didn't dictate, each with its why>
Concerns: <risks, smells, seams worth a follow-up>
Deferred: <explicitly not done, and where it should land>
```

Then send one line:

```bash
aoe send "ORCH_TITLE" "WORKER COMPLETE #<ISSUE_N>: packet at <absolute packet path>"
```

Then stop — the orchestrator archives your session. Concerns and Deferred are
what the follow-up issues are built from, so be specific. What happens beyond
your issue is not your concern.

**Fail** the same way when you cannot finish: the packet's first line is
`WORKER FAILED #<ISSUE_N>`, followed by what stopped you, what is committed
and pushed, and what a human needs to decide; the message is
`WORKER FAILED #<ISSUE_N>: packet at <absolute packet path>`.

## Handoff (~60% context)

After committing and writing the handoff document into the worktree root:

```bash
aoe send "ORCH_TITLE" "WORKER HANDOFF #<ISSUE_N>: handoff at <absolute path>"
```

The handoff document's first line states where the work stands and what's
next. Then stop — the orchestrator spawns a continuation session with
`--resume`.
