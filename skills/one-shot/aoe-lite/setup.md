# Setup

Produce one file: `~/.pf-aoe/<KEY>/playbook.md`. Everything else in the repo stays as you found it.

## 1. Learn how the project operates

If a playbook already exists, read it and update it instead of starting over.

Read what the repo offers: `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING`, the README, ADRs and domain glossaries, PR templates, CI config, commit hooks, task runners, `git log`, recent merged PRs, and open issues with their labels. Note the source of each finding.

## 2. Ask about what the repo can't tell you

Draft the playbook, then ask only about what's still unknown or is a matter of preference. Give a recommendation with each question. These usually need asking:

- Where work comes from, and whether you may create issues or labels here
- Whether stacked PRs and force-pushes are acceptable to the team, and whether PRs start as drafts
- Which worker command to use
- How many workers may run at once. Recommend 1 unless worktrees can run side by side without sharing ports, databases or containers
- What review each piece gets before its PR
- When to pause for you, and when to keep going

## 3. Write it

Show the draft, apply corrections, and write it after the user approves. Use this shape, and add or drop lines to fit the project:

```markdown
# <project> playbook

## How the project operates
- Work comes from: <issues with label X / a board / plan files> (source)
- Branches: <format> (source)
- Commits: <format> (source)
- PRs: <template, draft or ready, required reviewers> (source)
- Checks before a PR: <exact commands> (source)
- Merging: <method and exact command, who merges> (source)
- Worktrees: <setup and teardown, if the default sibling folder doesn't suit>
- Decision records: <paths to ADRs and glossary, or none> (source)
- Things to watch for: <unwritten rules, risky areas>

## My preferences on top
- Worker command: <e.g. claude --model opus --permission-mode auto>
- Parallel workers: <n>
- Stacked PRs: <allowed or not>
- Review before each PR: <e.g. a fresh subagent reviews the diff>
- Pause for me when: <...>
- Keep out of the repo: <e.g. personal notes, and any mention of this playbook>
```

Done when the file exists and the user has approved it. Finish by giving its path and these allow rules for `~/.claude/settings.json`. The first two stop the orchestrator prompting each time, and the third stops auto mode blocking worker spawns: `Read(~/.pf-aoe/**)`, `Edit(~/.pf-aoe/**)` and `Bash(aoe add *)`.
