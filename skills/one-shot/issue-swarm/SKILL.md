---
name: issue-swarm
description: >
  Pick up to N uncompleted issues of a PRD (from the issue tracker or a local
  issues file, per the repo's issues mode) and implement them in parallel — one isolated subagent per issue, each in its own git worktree,
  gated by a reviewer subagent before being marked done. Use when the user wants
  to run several issues at once, mentions "swarm the issues", "work in parallel",
  or wants autonomous fan-out implementation.
metadata:
  version: 0.9.0
---

<what-to-do>

Orchestrate parallel implementation of up to `N` open issues of a PRD. You are the coordinator. Real work happens in subagents you dispatch. You do not edit code yourself.

Run these phases in order:

1. **Select issues.**
2. **Set up isolated workspaces and shared exploration notes.**
3. **Dispatch implementer subagents in parallel.**
4. **Dispatch reviewer subagent per finished implementation.**
5. **Loop implement↔review per issue until pass or cap — reusing the same two agents.**
6. **Reconcile: merge (implementer syncs on conflict), gate the wave, integration-review, clean up.**
7. **Re-enter Phase 1 if newly-unblocked issues exist.**
8. **Report.**

Each phase is detailed below.

</what-to-do>

<supporting-info>

## Parameters

Defaults if user did not specify:

- `max_parallel` — 5
- `max_cycles` per issue — 3 (one implement + up to 3 reviewer-driven revisions)
- `branch_prefix` — `swarm/`
- `base_branch` — current branch at session start

Ask the user only if you cannot infer the issues file or base branch.

## Phase 1 — Select issues

1. Find the PRD location. Read `docs/agents/prds.md`, take the `**Location:**` value. Default `prds/`.
   **Resolve the issues mode.** Read the `**Issues:**` line in `docs/agents/prds.md`: `tracker` or `file`. With no such line, it's `tracker` if `docs/agents/prds.md` says in prose that issues live on the tracker (a legacy override) or `docs/agents/issue-tracker.md` names GitHub or GitLab, and `file` otherwise. In `tracker` mode, run every issue operation the way `docs/agents/issue-tracker.md` describes (GitHub via `gh` when that file is absent); no local `*-issues.md` file exists.
2. Find the issues. `file`: find `*-issues.md` files; if multiple, ask which. `tracker`: list open issues carrying a `prd:<feature>` label; if more than one PRD label has open issues, ask which.
3. Parse issues in dependency order (`tracker`: ascending issue number).
4. For each, check `git log --all --grep="^Implements issue: <title>"` (`tracker`: `--grep="^Implements issue: #<n> "`). Skip if already implemented.
5. Skip any issue whose declared dependencies are not yet implemented (`tracker`: any blocker, native link or `Blocked by:` line, that is neither closed nor implemented).
6. Take the first `max_parallel` remaining. If zero, stop and report.

## Phase 2 — Workspaces

Once per run, before creating any worktree, record the starting point of `base_branch`:

```bash
run_start_sha=$(git rev-parse <base_branch>)
```

The integration review in Phase 6 diffs against it.

For each selected issue:

```bash
branch="swarm/issue-<slug>"
worktree=".worktrees/<slug>"
git worktree add -b "$branch" "$worktree" "<base_branch>"
```

Always worktree. Never edit in the main checkout. If a worktree path already exists, append a numeric suffix.

Record `{ issue_title, branch, worktree_path, cycles_used: 0, status: "implementing", implementer_agent: "impl-<slug>", reviewer_agent: "review-<slug>" }` in an in-memory state table. Print it.

The two agent names are how Phase 5 reaches back into the same agents instead of spawning fresh ones. If a worktree path needed a numeric suffix, give both names the same suffix so they stay unique within the session.

### Shared exploration notes

Isolated implementers each re-explore the same codebase from scratch. Pay that cost once per wave: dispatch one exploration subagent before Phase 3 and hand every implementer and reviewer a pointer to what it wrote.

```bash
notes_dir="$(git rev-parse --path-format=absolute --git-common-dir)/swarm-notes/$(git rev-parse --short "$run_start_sha")"
```

The directory sits outside every working tree (never committed) yet is reachable from all of them. The exploration subagent receives the wave's issue titles and bodies plus `notes_dir`, and writes one `<slug>.md` per issue:

- The files and modules the issue will touch, with file:line pointers.
- The **house sibling** for each artefact the issue will add: an existing entity, model, util, or test of the same kind to copy the shape from. Phase 4's conventions checklist checks against these.
- The test command and any setup it needs.
- Gotchas found on the way: hidden coupling, a generated file, a helper that already does half the job.

Notes are pointers, not copies: cite paths and lines, never paste code. Skip this step when every issue in the wave is small and self-locating. On a Phase 7 re-entry, run it again for the new wave's issues only; earlier notes stay.

## Phase 3 — Dispatch implementers in parallel

Send all implementer subagents in a single tool-call batch (parallel dispatch). Dispatch each with `name: "<implementer_agent>"` from the state table — Phase 5 revisions go back to this same agent, so it must be addressable.

Each implementer subagent receives:

- The issue title and full body — copy-paste from the issues file (`tracker`: from the tracker's view of the issue).
- The path to its exploration notes (`<notes_dir>/<slug>.md`), when the step ran. Read them before exploring; explore further only where they fall short. The issue body and notes are the only context it gets.
- The absolute `worktree_path` to work in.
- The branch name.
- The repo's test command (read from `package.json`, `mix.exs`, etc.).
- Instructions:
  - Work only in the given worktree path.
  - Follow TDD in vertical slices: one failing test → minimal code to pass → next test. Never write all tests upfront then implement in bulk — bulk tests verify imagined behaviour, not real behaviour.
  - Test observable behaviour through the public interface, not internals. Expected values must be independent literals / worked examples / the spec — never recomputed the way the code computes them.
  - Refactor only once green, never while a test is red.
  - Mock only at system boundaries (external APIs, DB, time, fs). Never mock your own modules or internal collaborators.
  - Do not touch files outside the issue scope.
  - Do NOT touch the issues file, the PRD, the tracker issues, or anything under `docs/adr/`. Issue retirement and PRD cleanup are handled by the coordinator in Phase 6.
  - On done: stage code only. Commit once per round — one for this initial pass, one more for each Phase 5 revision — using the format in "Commit format" below. Never amend an earlier commit. Return JSON `{"status":"done","commit":"<sha>","summary":"..."}`.
  - On block: return `{"status":"blocked","reason":"..."}`.

Implementer prompts must be self-contained. Do not pass session history.

## Phase 4 — Dispatch reviewers

When an implementer returns `done`, dispatch a reviewer subagent for that issue (parallel with other reviewers/implementers in flight). Dispatch it with `name: "<reviewer_agent>"` from the state table — re-reviews in Phase 5 go back to this same agent.

Reviewer receives:

- Issue title + body.
- The exploration notes path, when the step ran: the house siblings it lists are the first comparators for the conventions checklist.
- Worktree path.
- `git diff <base_branch>...<branch>` as context, or instruction to compute it.
- Test command.
- Instructions:
  - Run the test suite in the worktree. Paste actual output.
  - Read the diff. Check: does it implement the issue, avoid scope creep, include tests?
  - Apply the **conventions checklist** below. "Follows existing patterns" is too vague to enforce — a subagent with no codebase memory reinvents what already exists.
  - Apply the **adversarial checklist** below. Pattern/scope review is not enough — the misses that reach production are runtime, contract, and edge-case bugs that read fine locally.
  - Return JSON: `{"verdict":"approve"|"request-changes","tests_passed":true|false,"notes":"..."}`.
  - Reviewer never edits code.

### Conventions checklist (Phase 4 reviewer)

An isolated implementer has no memory of the codebase, so it re-invents shapes that already have a house pattern. For every new artefact in the diff, find a **sibling of the same kind already in the repo** and diff against it — cite the sibling's file:line.

- **Find the sibling first.** New DB entity → open another entity. New model/DAO → open another model. New util → grep for one that already does this. Approving without having opened a comparator means you did not check conformance.
- **Match the type pattern.** Does the new code type its data the way siblings do (named domain type, `Record<keyof T, ...>`, generated spec types) rather than `Record<string, unknown>`, inline anonymous shapes, or `as`-casts? Hand-rolled casting where siblings use a typed/validated layer is a flag.
- **Reuse, don't reinvent.** Before accepting a hand-written loop, fan-out, retry, pagination, or mapper, confirm no existing util/helper already does it (`mapLimit`, shared DAO base, etc.). A bespoke `Promise.all` next to an established `mapLimit` is a flag.
- **Naming matches the domain.** New types/functions use the project's ubiquitous language, not invented jargon (e.g. coined "Mirror"/"Projection"/"Wrapper" suffixes the rest of the code never uses). If a sibling concept has a name, reuse it.
- **No AI-comment noise.** Narrating comments that restate the code (`// upsert the row`, `// loop over items`) are not house style — flag them for removal. JSDoc on public API is fine.

Any deviation without a stated reason → `request-changes` naming the sibling it should have matched.

### Adversarial checklist (Phase 4 reviewer)

Do not approve until each relevant item is actively checked — name the file:line you verified, not "looks fine."

- **Model the deployed runtime, not the code.** For anything that calls an external API, ask what it *actually returns today* (empty, partial, missing fields), not what the happy-path test feeds it. Trace one real upstream payload end-to-end.
- **Distrust self-confirming tests.** A test that builds its own input to match its own assertion proves nothing about production. If the only evidence a path works is a hand-fed fixture, treat the path as **unverified** and require a test that drives the real producer.
- **Reject test smells that pass vacuously.** Three specific ones → `request-changes`, not a soft note:
  - *Tautological* — expected value recomputed the way the code computes it (`expect(add(a, b)).toBe(a + b)`, a hand-derived snapshot). Break the code wrong and the assertion breaks wrong with it. Demand an independent literal.
  - *External-means* — asserts by reaching past the interface (querying the DB directly, poking internals) instead of the public API the caller uses. Couples the test to storage, survives no refactor.
  - *Implementation-coupled* — mocks internal collaborators or asserts call counts/order; breaks on refactor with unchanged behaviour.
- **Doc ≠ mitigation.** A known dangerous behaviour described in a comment, checklist, or markdown file is **not** mitigated. A doc cannot stop a deploy. Dangerous-by-default paths need a code-level guard (a default-off flag, an assertion, an early return) before approval — request changes if the only "fix" is prose.
- **Write-path interrogation.** For any DB/storage write reachable from an external event, force-answer:
  - Duplicate input? (repeated array entries → duplicate keys → storage-engine rejection)
  - Idempotent under at-least-once redelivery / retry after partial failure?
  - Any read feeding the write strongly consistent, or can lag/eventual-consistency orphan or resurrect rows?
  - Engine limits respected? (batch/transaction item caps, item size, key uniqueness)
  - Side-effect ordering under retry? (alerts/emails posted before the commit they describe re-fire on retry)
- **Concurrency & ordering.** If correctness assumes events for one key arrive in order or one-at-a-time, is that guaranteed upstream, or just hoped for? Unconditional writes that depend on ordering are a flag.
- **Design smells (over-abstraction).** An isolated implementer, told to "make it clean," tends to invent structure the issue never needed. Judge modules on depth (behaviour per unit of interface), not size.
  - *Deletion test* — mentally remove the new module/wrapper. If its complexity just vanishes and callers get simpler, it's a pass-through adding interface for no behaviour → `request-changes`.
  - *One adapter, no seam* — a new interface/abstraction introduced with a single implementation and no real, present variation is premature. Don't bless a seam until there are genuinely two things behind it.
  - Also flag shallow modules (large interface, little behaviour) and leaked internals (callers must know implementation details to use it correctly).

Any unchecked relevant item → `request-changes` with the specific question, not a vague note.

## Phase 5 — Cycle loop

Revisions reuse the agents from Phases 3 and 4 — `SendMessage` to `implementer_agent` and `reviewer_agent`, never a fresh `Agent` dispatch. A cold implementer re-reads the codebase and re-derives the design just to apply a review note; the one that wrote the diff already holds it. Same for the reviewer: one that remembers its own notes can tell whether they were actually addressed, rather than re-deriving a verdict from scratch.

For each issue:

- `approve` + `tests_passed: true` → mark `status: approved`. Done.
- Otherwise → increment `cycles_used`. If `< max_cycles`:
  1. `SendMessage` the reviewer notes to `implementer_agent`. It keeps its worktree and its prior work; tell it to amend nothing — commit the revision on top. Same return contract as Phase 3 (`done` / `blocked`).
  2. On `done`, `SendMessage` `reviewer_agent` to re-review, telling it which of its notes the revision claims to address. Same return contract as Phase 4.
  3. Loop.
- If `cycles_used == max_cycles` → mark `status: needs-human`. Stop touching this issue. Leave worktree + branch intact.

If `SendMessage` fails to reach either agent (killed, expired), fall back to a fresh `Agent` dispatch under the same name, carrying the full Phase 3/4 prompt **plus** the accumulated reviewer notes and `git log <base_branch>..<branch>` so the replacement is not blind to what already landed. Record the fallback in the state table notes — it costs a full cold start and is worth seeing in the Phase 8 report.

Track all dispatched agents. Continue until every issue is `approved`, `blocked`, or `needs-human`. (`conflict` is assigned later, during Phase 6 merge.)

## Commit format (implementer subagents)

```
<50-char present-tense subject>

Why:

* <one-line bullet on the need this change addresses>

This change addresses the need by:

* <one-line bullet on how the change addresses it>
* <side effects, if any>

Implements issue: <issue title>
```

In `tracker` mode the trailer carries the number, and a `Closes` line follows it:

```
Implements issue: #<n> <issue title>
Closes #<n>
```

Wrap body at 72 columns. The `Implements issue:` trailer is required — `git log --grep` uses it for next-issue detection in subsequent runs. Keep it even though the issue is later removed from the file by the coordinator.

## Phase 6 — Reconcile and clean up

Default behaviour: merge approved work directly into `base_branch`. No PRs.

For each issue in the state table:

- **approved**: integrate into `base_branch`:

  ```bash
  git checkout <base_branch>
  git merge --ff-only <branch> || (
    git checkout <branch> &&
    git rebase <base_branch> &&
    git checkout <base_branch> &&
    git merge --ff-only <branch>
  )
  ```

  On rebase conflict, abort the rebase (`git rebase --abort`) and hand the sync to the issue's own implementer: it wrote the diff, so it holds the design a conflict has to be resolved against, which a cold resolver would have to re-derive.

  1. `SendMessage` `implementer_agent`: in its worktree, merge the current `<base_branch>` tip into its branch (`git merge <base_branch>`; a merge, not a rebase, so no commit is rewritten), resolve each conflict by the intent of both sides, run the test suite, and return the Phase 3 contract with the merge commit's sha.
  2. On `done`, `SendMessage` `reviewer_agent` to review the merge commit only (`git show <sha>`): resolutions that drop either side's intent → `request-changes`. Same return contract as Phase 4.
  3. On `approve` + `tests_passed: true`, retry `git merge --ff-only <branch>`; it now fast-forwards.

  This sync is one attempt and does not consume a `max_cycles` round. If the implementer returns `blocked`, the reviewer requests changes, or the agent is unreachable (fall back as in Phase 5), mark the issue `conflict`, leave the branch + worktree intact, and append to `swarm-needs-human.md`.

  On successful merge: `git branch -d <branch>` and `git worktree remove <worktree_path>`.

- **needs-human**, **blocked**, **conflict**: leave worktree + branch in place. Append issue title to `swarm-needs-human.md` at repo root with reviewer or failure notes.

- After all per-issue work: `git worktree prune`.

### Wave gate (after the wave's merges)

Branches that were green in isolation can be red together — semantic conflicts survive a clean rebase (issue A renames a helper issue B calls, two issues each add a migration with the same number). After merging the wave's approved branches, dispatch one gate subagent to run the full test suite on `base_branch` in the main checkout and return `{"passed":true|false,"output":"<failing excerpt>"}`.

If the gate fails:

- Do not attempt automated cross-issue fixes.
- Append the failure (with test output) to `swarm-needs-human.md`.
- Skip Phase 7 re-entry — a red `base_branch` must not seed new worktrees. Proceed to Phase 8 and report the gate failure prominently.

### Integration review (every ~3 merged issues)

Independently-implemented issues drift even when each passes review in isolation: duplicated helpers, divergent names for the same concept, contradictory patterns for the same kind of artefact. After every ~3 issues merged into `base_branch` — and once at run end if any merges happened since the last check — dispatch one integration-reviewer subagent with:

- `git diff <run_start_sha>..<base_branch>` (or instruction to compute it).
- The titles of the issues merged since the last integration review.
- Instructions: check **only cross-issue concerns** — duplication introduced by different issues, naming drift for the same domain concept, conflicting patterns between sibling artefacts. Do not re-review individual issues; Phase 4 already did.
- Return JSON: `{"findings":[{"kind":"duplication"|"naming-drift"|"pattern-conflict","files":["..."],"note":"..."}]}`.

Findings never block or revert merges. Append them to `swarm-needs-human.md` and include them in the Phase 8 report as follow-up work.

### Issues-file and PRD cleanup (coordinator only)

After all merges in Phase 6 complete, the coordinator does the bookkeeping in a single commit on `base_branch`:

1. `file`: remove every approved-and-merged issue from the `*-issues.md` file. `tracker`: leave the tracker alone — each merged commit's `Closes #<n>` closes its issue when it reaches the default branch.
2. If no issues remain — neither implemented nor pending (`tracker`: every `prd:<feature>` issue is implemented or closed) — also delete the parent PRD file, and the issues file in `file` mode.
3. Stage and commit:

```
Clean up implemented issues for <feature>

Why:

* Implemented issues are no longer needed — the feature is its own documentation.

This change addresses the need by:

* Removing <N> implemented issue(s) from <issues-path>. (`file` mode only)
* <If applicable> Deleting <prd-path> (and <issues-path> in `file` mode) now that all issues are implemented.
```

Skip this commit entirely if zero issues were approved this run, or if it would be empty (`tracker` mode with issues still pending).

Do NOT touch ADRs under `docs/adr/`. Those are managed by the grilling session.

If a remote exists and the user opted in (skill arg `push: true`), push `base_branch`. Default is local only — no push, no PR.

## Phase 7 — Re-enter Phase 1 if newly-unblocked issues exist

After Phase 6 finishes, dependencies that were blocked may now be satisfied (their prerequisite issues just got merged with `Implements issue:` trailers).

Re-run Phase 1 selection against the (possibly updated) issues file:

- If it returns one or more selectable issues → restart from Phase 2 with those issues. Keep accumulating results into the same state table.
- If it returns zero → proceed to Phase 8.

Stop conditions for the outer loop (any one ends it):

- The Phase 6 wave gate failed (red `base_branch`).
- Phase 1 returns zero selectable issues.
- Every remaining open issue is `blocked`, `needs-human`, or `conflict` (no forward progress possible).
- The previous outer iteration approved zero issues (prevents infinite spin on a stuck dependency graph).

Carry forward the full state table across iterations so Phase 8 reports every issue touched across all passes.

## Phase 8 — Report

Print a table:

| Issue | Status | Cycles | Branch | Worktree | Notes |

Then a one-line summary: `Approved X / Needs-human Y / Blocked Z out of N`.

Then remove `notes_dir`: it served this run only.

## Constraints

- You are the coordinator. You never run `Edit`, `Write`, or repo test commands directly. Everything goes through subagents.
- All subagents get isolated context. No session history leakage. This bans coordinator/session history, not an agent's own memory of the issue it is working — reusing one agent across that issue's revision cycles is the point of Phase 5, not a violation.
- Never operate in the main checkout. Always a worktree.
- Permission mode is not your concern. Do not gate behaviour on auto/bypass/normal.
- If `max_cycles` is hit, stop on that issue. Do not invent new strategies. Human will pick it up.
- Never force-push, never amend.
- Delete branches only after a successful fast-forward merge into `base_branch`. Otherwise leave them.
- Default cleanup = merge to `base_branch`, no PR. Only open PRs if the user explicitly asks.
- One implementer per issue for the whole run, reused across cycles. Never two implementers on the same issue at once, and never a second implementer agent for an issue that already has a live one — `SendMessage` it instead.

## Failure modes to watch

- Worktree path collision → suffix and retry once, else mark blocked.
- Implementer returns `done` but no commit exists → treat as blocked.
- Reviewer cannot run tests (missing deps) → reviewer reports `tests_passed: false` with reason. Counts as request-changes.
- `SendMessage` cannot reach a named agent in Phase 5 → cold-start fallback per Phase 5, noted in the state table. Do not silently spawn an unnamed replacement; the name is what keeps the next cycle addressable.
- Agent name already taken this session (an earlier outer-loop pass used the same slug) → suffix both names as in Phase 2 rather than reusing an agent that belongs to a different issue.
- All issues already implemented → stop in Phase 1, print "nothing to do".

</supporting-info>
