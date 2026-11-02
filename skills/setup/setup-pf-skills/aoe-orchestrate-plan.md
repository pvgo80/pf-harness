# aoe orchestrate plan

Repo-specific overrides for `aoe-orchestrate-plan` constants. Only the fields
present here override the skill's built-in defaults; delete a line to fall
back. How to work in this repo (gates before commit, coding standards) belongs
in `CLAUDE.md` / `AGENTS.md`, not here.

**WORKER_CMD:** `<command that starts a worker session, e.g. claude --model opus --permission-mode auto>`

**MAX_PARALLEL:** `<how many spoke sessions may run at once, e.g. 3 — only once worktrees can run side by side>`

**MERGE_CMD:** `<how the PRs merge at close-out, in base order, e.g. just merge-stack <PR_N…>>`
