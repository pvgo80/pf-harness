---
name: aoe-orchestrate-plan
description: >
  A resident orchestrator session works a group of GitHub issues sharing a label as stacked PRs — one
  aoe worker session per issue, independent lanes of the dependency graph in
  parallel, restack workers giving multi-parent issues a single base, workers
  reporting completion packets back to the hub instead of spawning each other,
  HITL/AFK mode switchable mid-run, and a close-out that synthesizes all worker
  feedback, plans follow-up issues, and merges and cleans up on the human's
  yes. Requires the aoe session manager. Use when the user asks to
  "orchestrate the plan", "run the orchestrated stack", "run the stacked
  pipeline", or "work the tagged stack" on an issue tag (e.g. prd:billing).
metadata:
  version: 0.2.0
disable-model-invocation: true
---

# aoe orchestrate plan

Work every open issue carrying a label (e.g. `prd:billing`) as stacked PRs,
coordinated by a **resident orchestrator session** — the hub. The issues'
dependency graph becomes a **forest of stacks**: each issue branches off the
one issue it builds on, and independent **lanes** run in parallel, up to
`--parallel` sessions at once. Spokes are sessions with one narrow job:

- a **worker** owns exactly its issue, its worktree, and its PR;
- a **restack worker** gives a multi-parent issue a single base by rebasing
  its finished parents into one line.

Every spoke's last act is a completion packet back to the hub. The
orchestrator holds everything cross-cutting: the graph and the ledger, the
HITL/AFK mode, the packets, and the close-out: synthesis, follow-up issues,
merge, and cleanup. Spokes never select issues, never spawn sessions, and
never learn what comes after them.

Review happens per generation: every commit gets an **adversarial subagent
review** first (same model, in-session — issue-worker's gate). There is no
end-of-stack review. What a worker could not fix in its slice travels in its
packet and comes out the other end as a follow-up issue.

## Mode — read your protocol

| Flags                        | You are             | Read                                   |
| ---------------------------- | ------------------- | -------------------------------------- |
| neither `--worker` nor `--restack` | the orchestrator | [orchestrator.md](orchestrator.md) |
| `--worker`                   | a worker            | [worker.md](worker.md)                 |
| `--restack`                  | a restack worker    | [restack.md](restack.md)               |

Read this file, then your protocol file — only your own. Everything below
applies to every session.

## Arguments

`/aoe-orchestrate-plan <tag> [--order <n1,n2,…>] [--limit <N>] [--parallel <K>] [--mode afk|hitl] [--base <branch>] [--stack-base <branch>]`

- `<tag>` (required): the issue label that groups the work, e.g. `prd:billing`.
- `--order <n1,n2,…>`: scope and priority. Issues not listed are out of scope
  for this run; among ready issues, listed order breaks ties. It never
  overrides a dependency.
- `--limit <N>`: work at most N issues this run, then close out.
- `--parallel <K>`: at most K spoke sessions in flight; defaults to
  MAX_PARALLEL.
- `--mode afk|hitl`: starting mode (default `afk`). See orchestrator.md.
- `--base <branch>`: base branch for every lane root; defaults to the repo
  default branch.
- `--stack-base <branch>`: the branch the whole forest merges into; defaults
  to the repo default branch.

Spoke flags, **set by the orchestrator — humans never type them**:

`/aoe-orchestrate-plan <tag> --worker --issue <N> --base <branch> --stack-base <branch> --orchestrator <title> [--notes <path>] [--resume]`

`/aoe-orchestrate-plan <tag> --restack --for <N> --line <n1,n2,…> --orchestrator <title> [--notes <path>]`

- `--orchestrator <title>`: the hub session's title — the reply address for
  every packet.
- `--notes <path>`: a file under MSG_DIR the orchestrator wrote for this
  spoke. Read it before anything else.
- The rest are explained in the spoke's protocol file.

## Prerequisites — validate before anything else

- `command -v aoe` succeeds. If not, halt: this skill orchestrates exclusively
  through the aoe session manager; without it there is nothing to run.
- `gh auth status` succeeds as a single known user.
- At least one open issue carries `<tag>` (orchestrator mode only).
- A repo permission allowlist covering the pipeline's routine commands keeps
  approval pauses rare — the `fewer-permission-prompts` skill generates it.
  Missing allowlist ≠ halt, but expect more pauses in afk runs.

## Session rules

**Sessions always start in the repo's main checkout root** and work in a
worktree from there. Never launch a session inside a worktree path — the root
is already trusted and carries the repo's existing permission config; a new
path re-triggers Claude Code's startup gates and kills unattended panes.

### Permission posture

Spokes launch with `--permission-mode auto` and **never** with `aoe add -y`
(that launches `claude --dangerously-skip-permissions`). In auto mode a
safety classifier auto-approves routine actions and prompts only on ones it
judges risky or cannot assess. The repo's `.claude/settings.json` allowlist
still matters: allowlisted commands skip the classifier and its latency. A
command that isn't auto-approved does not kill the pane — the session
**pauses** on the prompt, visible in the aoe dashboard, until a human
attaches and answers. Treat a paused session as "waiting for approval", never
as a failure. On an older Claude Code whose `--permission-mode` lacks `auto`,
fall back to `acceptEdits`.

**AFK mode does not change this posture.** Overnight autonomy comes from the
allowlist, not from loosening permissions. Full bypass is opt-in only: the
human asks in so many words, no `.claude/auto-mode-disabled` marker exists at
repo root or `~/.claude/`, and
`grep -q '"bypassPermissionsModeAccepted": *true' ~/.claude.json` succeeds
(grep only this key; the file is credential-bearing). If absent, halt and
tell the human to run `claude --dangerously-skip-permissions` once, accept,
`/exit` — never write the flag yourself.

### Liveness rule (every spawn)

After `aoe add`, wait ~20s and verify the session is alive
(`aoe session show "<title>"`, then `aoe session capture "<title>"` to read
its pane) BEFORE `aoe send` and before reporting anything — a message sent to
a dead pane is silently lost. Three states: **alive** → proceed; **paused on
a permission prompt** → alive, report "waiting for approval — attach with
`aoe session attach <title>`"; **dead** → remove the corpse session, capture
the pane's last output, report the matching remedy, and stop.

### Message rule (every `aoe send`)

`aoe send` types the message into the pane, and Claude Code turns a long or
multi-line input into a paste block. A slash command inside a paste never
runs — and the skill is slash-only, so a spoke can't load it any other way —
and the receiving session treats the block as someone else's text rather
than its user's instructions, so it may stop and ask. Parallel spokes also
type into the same orchestrator pane, and two long sends can interleave.

So every message is **one short line**: the slash command alone, or one
sentence naming a file to read. Everything longer — notes for a spoke,
packets for the hub — is a file under MSG_DIR, and the message passes its
absolute path.

## Constants (repo can override in `docs/agents/aoe-orchestrate-plan.md`)

Each constant below is a named field a repo can set in that file
(`**MERGE_CMD:** …`); a field that's absent keeps the default here. How to
work in the repo — gates before commit, coding standards — belongs in the
repo's `CLAUDE.md` / `AGENTS.md`, which every worker reads anyway.

- WORKER_CMD: `claude --model opus --permission-mode auto` — launches every
  spoke. When the command goes through a launcher wrapper (e.g.
  `ollama launch`), put Claude Code's flags after `--`, or the pane dies at
  launch on an unknown-flag error.
- MAX_PARALLEL: `1` — the default for `--parallel`. Raise it once the repo's
  worktrees can run side by side without fighting over ports, databases, or
  containers (`docs/agents/worktrees.md` says how the repo isolates them).
- MERGE_CMD: how one PR merges, run at close-out in base order (each PR
  after the one its branch is based on) only after the human's yes. Default,
  per PR:
  `gh pr edit <PR_N> --base STACK_BASE && gh pr merge <PR_N> --merge`. The
  merge commit keeps each upper PR's diff to its own slice once retargeted; a
  repo that squashes or has its own stack tooling (a `just merge-stack`
  recipe, `gh stack merge`) names it here, taking the PR list in that order.
- MSG_DIR: `$(git rev-parse --path-format=absolute --git-common-dir)/aoe-orchestrate`
  — inside `.git`, so never committed and visible from every worktree.
- FEATURE: the tag with its prefix stripped up to the first `:`
  (`prd:billing` → `billing`).
- REPO: the repository's **main checkout path** — every session starts there.
- ORCH_TITLE: `<Feature> - Orchestrator` unless overridden by
  `--orchestrator`.
- Branch naming: `<FEATURE>/issue-<ISSUE_N>` by default; if the repo's
  worktree tooling requires branch == directory name, use
  `<FEATURE>-issue-<ISSUE_N>`. Either way the branch carries
  `issue-<ISSUE_N>`.

## Placeholders — two different numbers, never conflate them

- **ISSUE_N** — the GitHub issue number. Used in branch names, session
  titles, and the `Closes #` trailer.
- **PR_N** — the PR number returned by `gh pr create`. Used in PR commands.
  It is NOT the issue number.

## Invariants

- The orchestrator is the only spawner of sessions and the only reader of the
  graph; at most `--parallel` spokes are in flight.
- Every session starts in the main checkout root — never inside a worktree.
- Every `aoe send` is one line; anything longer travels as a file under
  MSG_DIR.
- Every event flows through the orchestrator: completion, handoff, failures.
  Spokes know their reply address and nothing beyond their job.
- Every commit and every conflict resolution passes an adversarial subagent
  review first.
- A branch is **frozen** once any branch is based on it. Only a restack
  worker rewrites a branch, and only one nothing is based on. Rewriting a
  frozen branch is the human's call — escalate, never quietly restack.
- Merging happens only at close-out, on the human's explicit yes, in both
  modes — spokes never merge, and the orchestrator never merges mid-run.
  AFK changes pacing, never permissions and never merge rights.
- A spoke session is archived the moment its packet arrives and purged at
  cleanup.
- A failure holds the failed issue and everything downstream of it; it never
  skips them.
