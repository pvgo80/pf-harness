# Restack protocol (`--restack`)

You are one spoke, running in the main checkout root. Issue C depends on
several parents that were built in parallel; a branch has one base, so you
stack the parents into one line and C will branch off its top. Your
jurisdiction is exactly: the **tops** of the line — their branches and PRs —
and your packet. You never touch the bottom, never touch C, never write new
features, never spawn sessions, never merge. Establish coordinates:

- C: from `--for`. ORCH_TITLE: from `--orchestrator` — your reply address.
- LINE: from `--line`, bottom first: `L0` is the **bottom** and stays as it
  is; `L1…Lk` are the **tops**, rebased in order, each onto the one before.
- For each issue in LINE: its branch (Constants branch naming) and its PR
  (`gh pr list --head <branch> --json number,baseRefName`).
- The notes file (`--notes`) lists every parent's packet. Read them all
  before step 1 — the parents' independent decisions are where the conflicts
  come from, and their packets say what each side meant.

## 0. Check the line is safe to rewrite

`git fetch origin`. Every LINE branch exists on origin with an open PR. For
every top, nothing is based on it: `gh pr list --base <top branch>` is empty,
and `git branch -a --contains origin/<top branch>` lists no branch but the
top itself (this catches a child branched but not yet pushed or PR'd). A top
with something based on it is **frozen** — rewriting it is the human's call:
report it as a failure (step 4) and stop.

## 1. Rebase each top onto the line

For each top `Li`, in order, with NEW_BASE the branch of `L(i-1)`:

1. Work in `Li`'s existing worktree (`git worktree list` finds it; when none
   exists, `git worktree add ../<repo-name>-<FEATURE>-issue-<Li> <Li branch>`
   — the branch already exists). Make the local branch match origin:
   `git reset --hard origin/<Li branch>` (its worker pushed everything).
2. Record OLD_BASE (`git rev-parse origin/<Li's PR baseRefName>`) and OLD_TIP
   (`git rev-parse origin/<Li branch>`).
3. `git rebase --onto origin/<NEW_BASE> <OLD_BASE> <Li branch>` — this moves
   only `Li`'s own commits.
4. Resolve each conflict so **both** sides keep what their issue asked for.
   A resolution that needs a design choice neither issue made is not yours
   to make: **fail** (below).

## 2. Verify — gates and adversarial review

Run the repo's gates (per its `CLAUDE.md` / `AGENTS.md`) inside the
worktree: a clean rebase can still break behaviour on top of the new base.
Then dispatch an **adversarial reviewer subagent** — fresh agent, default
stance "broken until proven otherwise" — on the **resolution diff** only:
`git range-diff <OLD_BASE>..<OLD_TIP> origin/<NEW_BASE>..<Li branch>`, plus
both issues' packets and the test command. Fix as new commits on `Li`
(`fix: reconcile #<Li> with #<L(i-1)>`), re-review, up to 3 cycles. Still
failing → **fail** (below).

**Fail** means: put the unpushed top back exactly as it was
(`git rebase --abort` mid-rebase, else `git reset --hard <OLD_TIP>`), leave
the tops already published as they are, and report RESTACK FAILED (step 4).

## 3. Publish

```bash
git push --force-with-lease=<Li branch>:<OLD_TIP> origin <Li branch>
gh pr edit <Li PR_N> --base <NEW_BASE>
gh pr comment <Li PR_N> --body "Restacked onto #<L(i-1)> (<NEW_BASE>) to give #<C> a single base; #<Li> has no real dependency on #<L(i-1)>. Conflict resolutions: <each, or none>."
```

Then the next top. After the last one, `Lk`'s branch is the **tip**: C's
base.

## 4. Report — your last act

Write `<MSG_DIR>/<FEATURE>-restack-<C>-packet.md`:

```
RESTACK COMPLETE #<C>        (or: RESTACK FAILED #<C>)
Line: <L0 branch> ← <L1 branch> ← … (bottom first)
Tip: <Lk branch>
Rebased: per top — PR_N, old base → new base, pushed yes/no
Resolutions: per conflict — file, what each side wanted, what you kept, why
Concerns: <risks the reconciliation leaves, seams worth a follow-up>
Deferred: <what you could not reconcile, and where it should land>
```

A failure says why, which tops were already published (they stay rewritten),
and which were put back. Then send one line, and stop:

```bash
aoe send "ORCH_TITLE" "RESTACK COMPLETE #<C>: packet at <absolute packet path>"
# or: "RESTACK FAILED #<C>: packet at <absolute packet path>"
```
