---
name: setup-pf-skills
description: "Configure a repo for the pf-skills plugins: domain-doc layout, PRD/issue/plan locations, issue tracker and triage label vocabulary, and one-shot orchestration posture. Detects which plugins are installed and asks only the sections that apply. Run once per repo before first use of the building, engineering, or one-shot skills — or if those skills appear to be missing context about where to read or write files."
disable-model-invocation: true
---

# Setup pf-skills

Scaffold the per-repo configuration the pf-skills plugins assume. This is the
single merged replacement for several upstream installers
(including `setup-matt-pocock-skills`); it writes:

- **Domain docs** — where `CONTEXT.md` and ADRs live, and how skills consume them _(all plugins)_
- **PRDs and issues** — where `write-a-prd` writes, and whether the PRD lane's issues live on the tracker or in a local file _(one-shot)_
- **Issue tracker** — where issues live and how to operate on them _(engineering, one-shot)_
- **Triage labels** — the strings used for the five canonical triage roles _(engineering, only when `triage` is installed)_
- **One-shot orchestration** — post-implement ADR write target, plans location, auto-mode posture, base branch _(one-shot)_
- **aoe orchestrate plan** — worker session command, parallelism, and merge command for `aoe-orchestrate-plan` _(one-shot, only when the `aoe` CLI is on PATH)_

This is a prompt-driven skill, not a deterministic script. Explore, present
what you found, confirm with the user, then write.

## Process

### 1. Detect installed plugins

Which sections run depends on which pf-skills plugins are installed in this
repo. Check your available skills:

- **building** installed → `grill-with-docs` / `grill-me` / `prototype` are available
- **one-shot** installed → `one-shot` and its PRD lane (`write-a-prd` / `prd-to-issues` / `issue-worker` / `issue-swarm`) are available
- **engineering** installed → `to-tickets` / `to-spec` / `wayfinder` are available; note separately whether `triage` is available (it gates Section D)

If none of the three is installed, ask the user which plugin(s) they intend to
install here and configure for those; don't refuse.

Section gating: **A** always runs; **B** when one-shot; **C** when
engineering or one-shot (the PRD lane follows the tracker); **D** when
engineering and `triage` is installed; **E** when one-shot;
**F** when one-shot AND `command -v aoe` succeeds (an absent CLI means the
aoe pipeline can't run here, so it needs no config).

### 2. Explore

Look at the current repo to understand its starting state. Read whatever
exists; don't assume:

- `git remote -v` and `.git/config` — GitHub? GitLab? No remote? For one-shot, also the default branch (`git symbolic-ref refs/remotes/origin/HEAD` or `gh repo view --json defaultBranchRef`)
- `CLAUDE.md` and `AGENTS.md` at the repo root — does either exist? Is there already an `## Agent skills` section in either?
- `CONTEXT.md` and `CONTEXT-MAP.md` at the repo root
- `docs/adr/` and any `src/*/docs/adr/` directories — existing ADR convention + highest ADR number
- `docs/agents/` — has this skill (or an upstream setup skill) already run? Read whatever `domain.md` / `prds.md` / `issue-tracker.md` / `triage-labels.md` / `one-shot.md` / `aoe-orchestrate-plan.md` exist
- `prds/`, `docs/prds/`, `.scratch/prds/` — existing PRD location convention; `.scratch/` is also a sign the local-markdown issue tracker convention is in use
- `plans/`, `docs/plans/` — existing plan location convention (one-shot only)
- `.claude/auto-mode-disabled` (repo) and `~/.claude/auto-mode-disabled` (global) — is auto mode already vetoed? (one-shot only)
- Monorepo signals: `pnpm-workspace.yaml`, a `workspaces` field in `package.json`, or a populated `packages/*` with its own `src/`. Present only in a genuinely large multi-package repo; their absence means single-context, which is almost every repo.

### 3. Present findings and ask

Summarise what's present and what's missing. Then take the applicable sections
in order, **one at a time** — present a section, get the user's answer, then
move to the next. Don't dump them all at once.

Lead each section with the recommended answer so the user can accept it in a
word. Give a one-line explainer only when the choice genuinely branches; skip a
section entirely when gating (step 1) or exploration already settled it, and
confirm before reusing a `docs/agents/` file that already exists and looks
right.

**Section A — Domain docs.**

> Explainer: grilling and exploration skills read `CONTEXT.md` for this project's ubiquitous language and `docs/adr/` for past architectural decisions. They need to know whether the repo has one global context or several (e.g. a monorepo with separate frontend/backend contexts) so they look in the right place.

Default to **single-context** (one `CONTEXT.md` + `docs/adr/` at the repo
root); this fits almost every repo, so write it without asking. Offer
**multi-context** (a root `CONTEXT-MAP.md` pointing to per-context `CONTEXT.md`
files) only when exploration found monorepo signals — then confirm which layout
they want.

**Section B — PRD/issues location.** _(one-shot)_

> Explainer: `write-a-prd` writes `<location>/<feature>.md`; in `file` mode `prd-to-issues` writes `<location>/<feature>-issues.md` beside it (whether issues go to the tracker instead is settled in Section C). These are **scratch** — `issue-worker` deletes them once the last issue ships.

Default: if a `prds/` directory already exists, propose it; else default to
`prds/`. Choices:

- **`prds/`** (default) — top-level convention
- **`docs/prds/`** — alongside other docs
- **`.scratch/prds/`** — ephemeral / gitignored work
- **Custom path** — user-supplied

**Section C — Issue tracker.** _(engineering)_

> Explainer: The "issue tracker" is where issues live for this repo. Skills like `to-tickets`, `triage`, and `to-spec` read from and write to it, and so does the PRD lane (`prd-to-issues` / `issue-worker` / `issue-swarm`) unless you keep it on local files. They need to know whether to call `gh issue create`, write a markdown file under `.scratch/`, or follow some other workflow you describe. Pick the place you actually track work for this repo.

Default posture: these skills were designed for GitHub. If a `git remote`
points at GitHub, propose that. If a `git remote` points at GitLab
(`gitlab.com` or a self-hosted host), propose GitLab. Otherwise (or if the user
prefers), offer:

- **GitHub** — issues live in the repo's GitHub Issues (uses the `gh` CLI)
- **GitLab** — issues live in the repo's GitLab Issues (uses the [`glab`](https://gitlab.com/gitlab-org/cli) CLI)
- **Local markdown** — issues live as files under `.scratch/<feature>/` in this repo (good for solo projects or repos without a remote)
- **Other** (Jira, Linear, etc.) — ask the user to describe the workflow in one paragraph; record it as freeform prose

Record the choice in `docs/agents/issue-tracker.md`. The GitHub and GitLab
templates carry a "PRs as a request surface" flag, defaulted **off**. Leave it
off and don't raise it: a user who wants external PRs in the triage queue can
flip the flag in the file later.

**C2 — PRD-lane issues.** _(one-shot, only when C picked GitHub or GitLab)_

> Explainer: `prd-to-issues`, `issue-worker` and `issue-swarm` read one field, `**Issues:**` in `docs/agents/prds.md`. `tracker` puts the PRD's issues on the tracker you just picked (labelled `prd:<feature>`, with native blocking links, closed by `Closes #<n>` commits); `file` keeps them in a local `<location>/<feature>-issues.md` that never reaches the tracker.

Recommend **`tracker`**, since it matches the tracker the user just chose. Offer
`file` for repos that want cheap, local planning iteration. Record the answer
as the `**Issues:**` line in `docs/agents/prds.md`. When C picked local
markdown or "other", set `Issues` to `file` without asking: the PRD lane
only publishes to GitHub and GitLab.

**Section D — Triage label vocabulary.** _(engineering, only when `triage` is installed — an uninstalled skill needs no labels)_

Ask exactly one question:

> Do you want to keep the default triage labels? (recommended: **yes**)

The defaults are the five canonical roles, each label string equal to its name:
`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`.
On **yes**, write them as-is. Only if the user says no, usually because their
tracker already uses other names (e.g. `bug:triage` for `needs-triage`),
collect the overrides so `triage` applies existing labels instead of creating
duplicates.

**Section E — One-shot orchestration.** _(one-shot)_ Three decisions, still one at a time:

**E1 — ADR write location.**

> Explainer: one-shot's step-7 ADR-write phase writes new ADRs **post-implement**, numbered after the highest existing `NNNN-<slug>.md`. ADRs are the **durable record** — the pipeline never deletes them. This is usually the same `docs/adr/` the grilling phase reads from (Section A).

Default: `docs/adr/` (or the per-context `src/<context>/docs/adr/` layout if
multi-context). Confirm it matches Section A; offer a custom path only if the
repo splits read vs write ADR locations (rare).

**E2 — Plans location (single-PR lane).**

> Explainer: the single-PR lane reads its plan from `<location>/<feature>.md`. Plans are **scratch** — one-shot retires the plan via a committed deletion after ADRs land (step 7b). Multi-PR flows use PRDs/issues (Section B) instead.

Default: if a `plans/` directory exists, propose it; else default to `plans/`.
Choices: `plans/` (default), `docs/plans/`, `.scratch/plans/`, custom.

**E3 — Auto-mode posture.**

> Explainer: one-shot's `auto` mode runs the cascade unattended (commits, pushes, opens a draft PR). It's already gated at runtime by `--i-understand-the-risk` and hard caps (no merge, no force-push, no `--no-verify`, no writes to the default branch). A `.claude/auto-mode-disabled` marker at the repo root is a hard veto — one-shot refuses `auto` entirely and never silently downgrades.

Ask directly: **"Is this a shared or client repo where unattended commits/pushes could surprise teammates?"**

- **Yes → veto auto mode.** Write `.claude/auto-mode-disabled`. `hitl` + `grill` still work; `auto` refuses with the marker as the reason.
- **No → leave auto available** (default). No marker written. `auto` still requires `--i-understand-the-risk` at invocation time.

Don't write the marker by default — only when the user says it's shared/client.

**Section F — aoe orchestrate plan.** _(one-shot, only when `aoe` is on PATH)_

> Explainer: `aoe-orchestrate-plan` works a label-tagged GitHub issue group as stacked PRs: a resident orchestrator spawns one aoe worker session per issue (WORKER_CMD, default `claude --model opus --permission-mode auto`), runs independent lanes of the dependency graph in parallel (MAX_PARALLEL sessions at once, default 1), and, at close-out, merges the PRs in base order (each after the one it's stacked on) on the human's yes (MERGE_CMD, default: retarget each PR to the stack base, then `gh pr merge --merge`). Sessions run in supervised-autonomy mode — the auto-mode classifier approves routine actions, and anything else pauses for a human to attach and approve — so a repo allowlist (`fewer-permission-prompts`) keeps pauses and classifier latency rare.

Look for a merge convention first: a stack-merge recipe in the `justfile` /
`Makefile` / `package.json` scripts, a `scripts/merge-stack`-style helper, or
squash-only merges in the repo settings (`gh repo view --json
squashMergeAllowed,mergeCommitAllowed`). Squash-only breaks the default — say
so and propose a MERGE_CMD. Then check whether worktrees can run side by
side: `docs/agents/worktrees.md` assigning each worktree its own ports /
database / containers, or an app with no such shared resources, makes
MAX_PARALLEL above 1 safe — propose a value; otherwise keep 1.

Ask one question: **"Keep the orchestrator's defaults, or set a repo-specific
worker command, parallelism, and/or merge command?"** On keep (recommended unless the merge
check above found a reason not to), write nothing — the skill's built-in
defaults apply and no `docs/agents/aoe-orchestrate-plan.md` is needed. Only on
override, collect WORKER_CMD, MAX_PARALLEL, and/or MERGE_CMD and write the file.

### 4. Confirm and edit

Show the user a draft of:

- The `## Agent skills` block to add to whichever of `CLAUDE.md` / `AGENTS.md` is being edited (selection rules in step 5)
- The contents of each `docs/agents/*.md` file whose section ran
- Whether `.claude/auto-mode-disabled` will be written (Section E3)

Let them edit before writing.

### 5. Write

**Pick the file to edit:**

- If `CLAUDE.md` exists, edit it.
- Else if `AGENTS.md` exists, edit it.
- If neither exists, ask the user which one to create — don't pick for them.

Never create `AGENTS.md` when `CLAUDE.md` already exists (or vice versa) —
always edit the one that's already there.

If an `## Agent skills` block already exists in the chosen file, update its
contents in-place rather than appending a duplicate. Don't overwrite user edits
to the surrounding sections.

The block — include only the sub-blocks whose sections ran:

```markdown
## Agent skills

### Domain docs

[single-context | multi-context]. See `docs/agents/domain.md`.

### PRDs and issues

PRDs live in `<prd-location>/<feature>.md`; their issues [on the issue tracker, labelled `prd:<feature>` | in `<prd-location>/<feature>-issues.md`]. See `docs/agents/prds.md`.

### Issue tracker

[one-line summary of where issues are tracked]. See `docs/agents/issue-tracker.md`.

### Triage labels

[one-line summary of the label vocabulary]. See `docs/agents/triage-labels.md`.

### One-shot orchestration

Post-implement ADRs written to `<adr-location>`; single-PR plans in `<plans-location>`; auto mode [allowed | disabled via `.claude/auto-mode-disabled`]. Base branch: `<base>`. See `docs/agents/one-shot.md`.

### aoe orchestrate plan

Worker session command, parallelism, and/or merge command overridden for this repo. See `docs/agents/aoe-orchestrate-plan.md`.
```

(The aoe sub-block appears only when Section F wrote an override file — a
repo on the defaults gets neither the sub-block nor the file.)

Then write the `docs/agents/` files for the sections that ran, using the seed
templates in this skill folder as a starting point — substitute the chosen
layouts/locations before writing:

- [domain.md](./domain.md) — domain-doc consumer rules + layout (Section A)
- [prds.md](./prds.md) — PRD location and issues mode (Sections B + C2)
- [issue-tracker-github.md](./issue-tracker-github.md) / [issue-tracker-gitlab.md](./issue-tracker-gitlab.md) / [issue-tracker-local.md](./issue-tracker-local.md) — seed for `docs/agents/issue-tracker.md` (Section C); for "other" trackers, write it from scratch using the user's description
- [triage-labels.md](./triage-labels.md) — label mapping (Section D)
- [one-shot.md](./one-shot.md) — ADR-write location, plans location, auto-mode posture, base branch (Section E)
- [aoe-orchestrate-plan.md](./aoe-orchestrate-plan.md) — WORKER_CMD / MAX_PARALLEL / MERGE_CMD overrides (Section F; written only when the user overrides the defaults)

If the user chose to veto auto mode in E3, write an empty
`.claude/auto-mode-disabled` file at the repo root.

### 6. Done

Tell the user setup is complete and which installed skills now read from these
files. Mention they can edit `docs/agents/*.md` directly later, or delete
`.claude/auto-mode-disabled` to opt this repo back into auto mode — re-running
this skill is only necessary to switch tracker/layout/location decisions, or
after installing another pf-skills plugin whose section hasn't run here yet.
