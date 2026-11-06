# Run

You are the orchestrator. You spawn every worker, and every worker reports back to you. Your session title is `ORCH_TITLE`.

Read the playbook. If it's missing, tell the user to run `/aoe-lite setup` and stop. Also stop if `command -v aoe` fails.

`RUN_DIR` is `$(git rev-parse --path-format=absolute --git-common-dir)/aoe-lite/<run-slug>`. It's inside `.git`, so it's never committed and every worktree can see it. `REPO` is the main checkout path.

## The rules that always apply

These protect against real failures, and the playbook can't relax them.

- **One-line sends.** Every `aoe send` is one short line. A long message becomes a paste block, and parallel sends can interleave. Put anything longer in a file under `RUN_DIR` and send its absolute path.
- **Start in the main checkout.** Launch every session in `REPO`. A new path triggers startup prompts that stall unattended panes. Workers create their own worktrees from there.
- **Check before sending.** About 20 seconds after `aoe add`, run `aoe session capture "<title>"`. If the pane is alive, send. If it's waiting on a prompt, tell the user to run `aoe session attach "<title>"`. If it's dead, report its last output and remove the session.
- **Merge only on the user's yes,** in every mode. Workers never merge.
- **Keep permissions as configured.** A session paused on a prompt is waiting for approval, not failing. Use `aoe add -y` only if the user asks for it in so many words.

## 1. Plan the pieces

Work out the scope from the argument and the playbook's source of work. Split it into pieces that each make one PR, and note which pieces depend on others.

Show the pieces, their order and the base branch of each, then get the user's `go`. Skip this confirmation only if the playbook says to.

A piece that depends on an unmerged piece branches from it if the playbook allows stacked PRs. If not, it waits until the user has merged its parent.

## 2. Spawn workers

Run up to the playbook's number of parallel workers. For each piece, write a brief to `RUN_DIR/<piece>-brief.md`, then:

```bash
aoe add <REPO> -t "<Feature> - <piece>" -c '<worker command>' -l
aoe send "<Feature> - <piece>" "Read <absolute brief path> and follow it."
```

Then end your turn. Each worker's message wakes you, so there's no need to poll.

The brief holds everything the worker needs, because it may not have this skill or be able to read the playbook:

````markdown
# <piece>

You are one worker in a coordinated run. You own this piece, its worktree and its PR.

## The task
<what to build, with the issue link or text and any notes from earlier packets>

## How to work in this project
<the playbook's "How the project operates" and "My preferences on top", pasted in full>

## Setup
Base branch: <branch>. Create your branch and worktree from the main checkout, following the playbook. By default:
`git worktree add ../<repo>-<piece> -b <branch> <base>`

Keep every commit, PR and comment self-contained. Refer only to what others can open, so leave out this brief and the run folder.

## Decisions
Before designing, read the project's decision records, if it has any. In your packet, say whether each decision follows, extends or conflicts with them.

Decide anything local to this piece yourself, and record it with the alternatives you rejected. When a choice has costs outside this piece, such as a public contract change, narrowing a requirement, conflicting with a recorded decision, new domain terms, or deploy or rollback risk, build the least invasive option, mark it provisional, and list it under Trade-offs.

If your work includes a review, any change after the last passing review, hook-forced fixes included, gets re-reviewed before you commit.

## Finishing
Open the PR as the playbook describes. Leave merging to the user. Then write `<RUN_DIR>/<piece>-packet.md`:

```
<piece>: DONE | FAILED | HANDOFF
PR: <url>, branch, base
Shipped: what now exists
Decisions: choices the task didn't dictate, and why
Trade-offs: provisional choices for the user, with the options and the cost of each; empty if none
Deploy / compatibility: what breaks on deploy or rollback; empty if nothing
Concerns: risks and follow-ups
Next: anything needed to finish, if this is FAILED or HANDOFF
```

Use HANDOFF if your context is running low: commit, push, and say in Next what remains. Finally, run this one line and stop:

`aoe send "<ORCH_TITLE>" "<piece> finished: packet at <absolute packet path>"`
````

## 3. On each packet

Read the packet and act on its status:

- **DONE:** record it, then spawn whatever is now unblocked. Pass any concerns that matter into the next brief. Tell the user about any trade-offs now, and hold a dependent piece only if the choice would change its base.
- **HANDOFF:** spawn a continuation on the same branch, with a brief built from the packet.
- **FAILED:** hold that piece and anything depending on it, keep the rest going, and tell the user.

Run `aoe session archive "<title>"` once the packet is handled. If the user sends you a message mid-run, follow it.

When nothing is running or ready, close out.

## 4. Close out

1. Give the user a short digest: PRs in merge order, decisions they should know about, open trade-offs, deploy risks, concerns, anything held, and proposed follow-ups.
2. Write the digest to `~/.pf-aoe/<KEY>/runs/<date>-<run-slug>.md`, so it outlasts the run.
3. Ask about follow-ups and merging separately. Act only on a yes, using the playbook's merge method, and stop at the first merge that fails.
4. After merging, or if the user says to, remove the worktrees and sessions and delete `RUN_DIR`.

Done when the digest is written and the user has answered both questions.

If this run taught you something about the project, offer it as a playbook change.
