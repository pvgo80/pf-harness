---
name: aoe-lite
description: Lightweight aoe orchestration for any project, driven by one private playbook of how the project works and your preferences. Run `/aoe-lite setup` once per repo, then `/aoe-lite <what to work on>`.
disable-model-invocation: true
---

# aoe lite

A resident orchestrator session splits work into pieces and runs one aoe worker session per piece. The **playbook**, one private Markdown file per repo, says how the project operates and how you want to work on top of it. Where this skill is silent, the playbook decides; where both are silent, use judgement and record the choice in the worker's packet, flagging it as a trade-off if its cost reaches beyond the piece.

## Mode

| Argument | Read |
|---|---|
| `setup` | [setup.md](setup.md) |
| anything else | [run.md](run.md) |

## Playbook location

`~/.pf-aoe/<KEY>/playbook.md`, outside the repo. It survives a reclone and is shared by every worktree.

Work out `<KEY>` from the repo's remote:

1. Take the URL of `origin`, or of the only remote. With several remotes and no `origin`, ask which to use.
2. Normalise it to `host/path`: remove credentials, port and any `.git` suffix, and lowercase everything. `git@github.com:Acme/API.git` and `https://user:token@github.com/acme/api` both become `github.com/acme/api`.
3. With no remote, use `local/<folder-name>-<first 8 characters of the sha256 of the git common dir path>`.

Stop and report the problem if the result contains a `..` segment.
