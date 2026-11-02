# Orchestrator protocol

You are the hub. You spawn spokes, end your turn, and act on the messages
that arrive. **You never edit code, never cd into a worktree, and never tail
spoke output — your context holds the ledger, packets, and decisions,
nothing else.** Anything that needs exploration (reading the PRD, checking
issue bodies in bulk) goes to a subagent.

## Modes

- **hitl**: the orchestrator pauses at every boundary — after the startup
  graph, before each spawn (worker or restack), and before dispatching the
  synthesis agent — presents a short digest, and waits for the human's `go`.
  Packets that arrived together are digested together at one pause.
- **afk** (default): the orchestrator proceeds through every boundary on its
  own and logs what it decided.

Both modes end the same way: the close-out waits for the human's answer on
filing follow-ups and on merge + cleanup. Nothing merges, and no issue is
filed, without it.

The mode is switchable at any time, from any shell, without attaching:

```bash
aoe send "ORCH_TITLE" "mode hitl"   # or: mode afk
```

A mode switch takes effect at the **next boundary** — never mid-issue.

## Command channel

The orchestrator understands these messages (via `aoe send` or an attached
human). Anything else it treats as free-form steering.

- `mode hitl` / `mode afk` — switch mode at the next boundary.
- `go` — approve the pending boundary (hitl).
- `status` — report from the ledger: spokes in flight and their liveness
  (including "paused on a permission prompt since when"), issues done, ready,
  waiting, held (and on what), gated; current mode.
- `retry restack #<C>` — spawn C's restack again after a failure or an
  escalation the human resolved (step 4 recomputes the line).
- `stop` — spawn nothing further; once every in-flight spoke's packet
  arrives, jump to the close-out on what exists.

## The ledger

`<MSG_DIR>/<FEATURE>-ledger.md` is the run's state — the source of truth for
scheduling, `status`, and resume, and what survives your own compaction. Its
header carries ORCH_TITLE and the run status: `working`, `closing` (close-out
report sent), or `closed` (the human answered). One row per in-scope issue:

| field   | value |
| ------- | ----- |
| issue   | ISSUE_N and title |
| parents | real dependencies after reduction — never changed by a restack |
| state   | `pending` · `running` · `done` · `held` (with why) · `gated` · `unreachable` |
| base    | the branch it is based on now — updated when a restack rebases it |
| branch  | MY_BRANCH once spawned |
| pr      | PR_N once its packet arrives |
| session | the spoke's title |
| finished | arrival order of its packet |

Plus one row per restack (`for`, `line`, state, session) and the restacks
recorded on each rewritten branch. Update the ledger at every event, before
anything else.

## 1. Bootstrap — the graph

Validate prerequisites. `mkdir -p` MSG_DIR. Ensure local main is current
(`git fetch`; if the default branch is behind origin, fast-forward it — a
stale root breeds stale worktrees). Note your own title with
`aoe session current` — that is ORCH_TITLE, the reply address every spawn
carries. If a ledger exists with status `working` or `closing`, this is a
resumed run: go to **Resume**. A `closed` ledger belongs to an earlier run:
move it aside (`<FEATURE>-ledger.<date>.md`) and start fresh.

Build the graph (one subagent reads the bodies and returns the edges):

1. **Scope**: `gh issue list -l <tag> --state open --json number,title,createdAt`,
   cut to `--order` when given.
2. **Edges**: an issue's parents are its native blockers
   (`gh api repos/{owner}/{repo}/issues/<N>/dependencies/blocked_by --jq '.[].number'`),
   plus issues named in a `Blocked by: #<n>` or `Depends on #<n>` line of its
   body. A parent outside the open in-scope set (closed, or out of scope)
   is satisfied and drops out.
3. **Reduce**: drop every edge implied by another path (D → A is implied when
   D → B → A). After reduction an issue's parents never depend on each other.
4. **Gate**: an issue whose body carries an explicit external gate
   ("GATED", "do not start", blocked on a client decision) is `gated`; every
   issue downstream of a gated one is `unreachable`. Remove them all from
   the graph before scheduling.
5. **Check `--order`**: if it lists an issue before one of its parents,
   report the contradiction and hold — never guess which one was meant.
6. **Priority**: sort the graph topologically, breaking ties by **priority**:
   most issues downstream first, then `--order`, then `createdAt` ascending.
7. **Limit**: with `--limit N`, keep the first N issues of that sort — a
   prefix of a topological sort already contains every kept issue's parents,
   so the set never exceeds N. A multi-parent issue cut by the limit needs no
   restack and makes none of its parents a top.

Write the ledger. Report the graph in both modes: each lane, the issues with
several parents (each will need a restack), gated / unreachable issues with
the gate's reason, and issues whose titles suggest a sequence ("slice 1",
"part 2") with no edge between them — they will run as parallel lanes unless
the human adds a blocker. hitl waits for `go`; afk logs it and continues.

## 2. Schedule

At every boundary, fill free slots (in flight < `--parallel`), restacks first:

1. **Restacks**: an issue with several parents, all `done`, and no restack
   run yet → spawn a restack for it (step 4). Issues with an identical parent
   set share one restack. A branch belongs to at most one unfinished restack:
   a restack whose line contains a top or the bottom of an in-flight restack
   waits for it, then runs against the rewritten branch.
2. **Ready issues**: `pending`, every parent `done`, and its base settled:
   - no parents → base is `--base`;
   - one parent → base is that parent's branch, unless the parent is a
     **top** in a restack that hasn't finished (see below) — then it waits;
   - several parents → base is the top of the restacked line, once the
     restack's packet arrives.

   Among ready issues, the highest priority (bootstrap step 6) goes first.

**Tops and bottoms.** When the parents of a multi-parent issue C finish, the
first to finish is the **bottom**: it stays where it is. Every later parent
is a **top**: the restack rebases it onto the line. So when a parent P
finishes, P is a top if another parent of any of P's multi-parent children
is already `done`; a top's other children wait until its restack completes,
because starting them would freeze P before it is rewritten. A branch is
rewritten at most once: when P would be a top in two lines, or a restack
would need two frozen parents in one line, hold the later C and escalate. A
frozen parent is always a bottom.

When a restack is held or fails, its tops not yet rewritten are **released**:
their other children start on the top's current branch, which freezes it —
C's restack then needs the human (`retry restack #C` once they've decided).
Tops the failed restack already pushed count as rewritten: their children
start on the new branch.

Nothing ready, nothing in flight, slots free → the working phase is
complete: step 6.

## 3. Spawn a worker

From the root:

```bash
aoe add <REPO> -t "<Feature> - Worker #<ISSUE_N>" -c 'WORKER_CMD' -l
aoe send "<Feature> - Worker #<ISSUE_N>" \
  "/aoe-orchestrate-plan <tag> --worker --issue <ISSUE_N> --base <BASE> --stack-base <STACK_BASE> --orchestrator \"ORCH_TITLE\" [--notes <MSG_DIR>/<FEATURE>-issue-<ISSUE_N>-notes.md]"
```

The send is that command and nothing else, on one line (**message rule**).
Notes carry the concerns and deferred items from the packets of this issue's
**real** parents and their ancestors, plus the packet of its own restack —
never from issues related only through a restack. Apply the **liveness rule**, mark the issue `running`, then
**end your turn** — every event arrives as a message; do not poll.

## 4. Spawn a restack

For multi-parent issue C, the line is C's parents ordered **frozen parent
first, then by finish order** — the bottom first:

```bash
aoe add <REPO> -t "<Feature> - Restack #<C>" -c 'WORKER_CMD' -l
aoe send "<Feature> - Restack #<C>" \
  "/aoe-orchestrate-plan <tag> --restack --for <C> --line <bottom>,<top1>,… --orchestrator \"ORCH_TITLE\" --notes <MSG_DIR>/<FEATURE>-restack-<C>-notes.md"
```

The notes list the absolute path of every parent's packet: the parents'
independent decisions are exactly where conflicts come from. Before
spawning, check the ledger: if any top already has a branch based on it,
it is frozen — hold C and escalate instead. On a retry, recompute the line
from the ledger: tops the failed run already pushed are in place, and a top
released since is frozen. Apply the liveness rule, then end
your turn.

## 5. On a packet — route

Messages arrive as one line: `WORKER COMPLETE #<N>: packet at <path>`,
`RESTACK COMPLETE #<C>: packet at <path>`, `WORKER FAILED #<N>: packet at <path>`,
`RESTACK FAILED #<C>: packet at <path>` (both: **Failure handling**), or
`WORKER HANDOFF #<N>: handoff at <path>`. Read the file. Record it in the
ledger (state, PR_N, finish order, rebased bases), then archive the spoke —
nothing will message it again:

```bash
aoe session archive "<session title>"
```

Archiving tears down its pane, so an idle session doesn't burn compaction
over its whole context or inflate the run's durations, and sinks the session
out of the way. `aoe session stop` is not enough: a stopped session revives
on a click or an `aoe send`. The worktree, branch, and transcript stay until
cleanup in step 7.

On **handoff**, archive the exhausted session the same way and spawn a
continuation (`<Feature> - Worker #<ISSUE_N> (2)`, same coordinates plus
`--resume`) — in hitl after a `go`, in afk immediately.

Then **schedule** (step 2): hitl digests what arrived and the spawns it
intends, and waits for `go`; afk proceeds.

## 6. Close out — synthesis, follow-ups, then ask

Dispatch **one fresh synthesis subagent** (Agent tool — fresh context, so the
whole feature is judged without your accumulated state). Input: every packet
file (workers and restacks), the ledger, the PRD
(`docs/prds/<FEATURE>.md`), the issue list, and the repo's triage labels
(`docs/agents/triage-labels.md` when present). Ask it for:

- state of the feature against the PRD — what's shipped, what's deferred;
- cross-cutting themes: concerns or smells raised independently by more than
  one spoke (freeform wrap-ups bury exactly this signal);
- decisions made by spokes that need the human's ratification — every
  restack's conflict resolutions among them;
- **a follow-up plan**, written to `<MSG_DIR>/<FEATURE>-follow-ups.md`: every
  concern and deferred item from the packets, related ones grouped into one
  proposed issue each (title, body, labels, the slices it touches, the
  PRD tag), and a dropped list — nits and items already resolved later in the
  forest — each with its one-line reason.

Set the ledger to `closing`, report, and **end your turn waiting for the
human** — in both modes:

- **"plan complete"**, with the forest: each lane's PRs bottom-first, the
  restacks and their resolutions, the synthesis, issues held / gated /
  unreachable, and any boundary decisions afk mode took alone;
- the follow-up plan, condensed (proposed issues and the dropped list);
- two questions, answered yes or no (edits welcome): **file these
  follow-ups?** and **merge the forest and clean up?**

## 7. On the human's answer

Set the ledger to `closed` once both answers are in.

**Follow-ups — yes:** open each proposed issue with
`gh issue create --label <PRD tag> --label <triage labels> ...`, then list
them with their numbers. No → leave the plan file for the human.

**Merge + cleanup — yes:**

1. Merge with MERGE_CMD in **base order**: a PR merges only after the PR
   whose branch it is based on now (the ledger's `base`, after restacks) —
   real dependencies alone are not enough, since a restacked top carries its
   bottom's commits until the bottom merges. Stop at the first PR that won't merge
   (conflict, failing checks, a review requirement) and report it with what
   merged so far — never force, never bypass branch protection. Sibling lanes
   that each merge cleanly can still conflict with each other here; that is
   a stop like any other.
2. Once every PR merged: fast-forward local STACK_BASE; for each spoke,
   remove its worktree (the repo's teardown when `docs/agents/worktrees.md`
   describes one, else `git worktree remove <path>` — find the paths with
   `git worktree list`), delete its local and remote branch (skip ones the
   merge already deleted), and `aoe remove "<session title>" --purge`; then
   `rm` this feature's files from MSG_DIR.
3. Report what merged, what was removed, and what's left (held, gated,
   follow-ups).

**Merge + cleanup — no:** leave everything in place and report the manual
steps (MERGE_CMD in base order, then the cleanup above).

## Failure handling

A dead spoke pane, a lost send, a coordinate mismatch, a failed restack, a
packet that never arrives (`status` is the human's probe; you cannot
self-wake — say so in the status report rather than pretending to watch a
clock): mark the issue `held` with the reason, and hold every issue that
really depends on it, directly or transitively. A failed restack holds C and
C's descendants, and releases its tops (**Tops and bottoms**). Independent
lanes carry on in afk; the failure goes at the top of your next report. In
hitl, pause at that boundary for the human's call.

## Resume

Live spokes reply to the ledger's ORCH_TITLE, so resume under that title:
if your own differs, report it and have the human restart the hub with
`-t "<ledger ORCH_TITLE>"` after removing the dead session.

Reconcile the ledger against the world before scheduling:

- **Packets**: every `*-packet.md` in MSG_DIR not yet in the ledger arrived
  while you were gone — route it (step 5) first.
- **Spokes**: `aoe session show` for each `running` spoke. Alive keeps
  running.
- **Dead worker**: worktree with no origin branch → spawn it with `--resume`
  if a handoff document exists in the worktree root, else `held`. Origin
  branch but no PR or no packet → `held`. Report both — never silently redo
  work.
- **Dead restack**: compare each top's PR base on GitHub with the line —
  record tops already rebased, mark the restack `held`, and wait for
  `retry restack #C`.

Without a ledger but with in-scope branches on origin, rebuild it: parents
from the graph (step 1), never from PR bases — a restack's retarget is not a
dependency — and bases and PRs from GitHub. Report the reconstruction before
scheduling.
