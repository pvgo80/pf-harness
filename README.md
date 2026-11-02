# pf-harness

A curated catalog of Claude Code skills, packaged as a **plugin marketplace**. Skills are grouped into plugins, and each project opts in to exactly the plugins it needs. Nothing loads globally.

## Why

- **Minimal context per project.** Skills installed at user scope follow you into every project. Here, a fresh project starts with zero skills; each project's `.claude/settings.json` declares what it wants.
- **One version everywhere.** One catalog, updated deliberately, instead of the same plugin pinned at different commits across projects.
- **Curated, not mirrored.** Third-party skills are vendored (copied), so they can be pruned, adapted, or replaced without waiting on upstream.

## Plugins

| Plugin | What it does |
|---|---|
| `building` | Shared foundation both lanes start from: grilling interviews, prototyping, handoffs, knowledge capture, zoom-out context. Co-install with `engineering` or `one-shot`. |
| `engineering` | Tracker-based idea→ship lane: the `ask-the-flow` router, TDD, bug diagnosis, domain modelling, codebase design, spec/ticket flows, triage, research, code review, wayfinder planning, retros, wizard generation. |
| `one-shot` | Orchestrated plan→ship lane with three autonomy modes, a PRD lane (`write-a-prd` → `prd-to-issues` → `issue-worker`/`issue-swarm`), a review cascade (`code-review` → `code-fix`), commit/PR hygiene, and `aoe-orchestrate-plan` for stacked PRs over tagged GitHub issues. |
| `design` | Map the IA, define a design system, then build a navigable lo-fi wireframe. |
| `reviewing` | ADR decay sweeps, PRD drift checks, and multi-phase branch review. |
| `productivity` | Teaching, questionnaires, re-pitching unclear messages, writing for agents. |
| `marketing` | Humanizing AI-drafted copy. |
| `setup` | `/setup-pf-skills`: per-repo wizard that writes the `docs/agents/` config the `building`, `engineering`, and `one-shot` plugins read. Run once per repo. |

See [docs/catalog-map.md](docs/catalog-map.md) for how the flow-carrying plugins relate.

## Usage

Register the marketplace once (user scope is fine — registration itself loads no context):

```bash
claude plugin marketplace add pvgo80/pf-harness
```

Then, per project, install only what that project needs:

```bash
claude plugin install building@pf-skills --scope project   # recorded in .claude/settings.json (committable)
claude plugin install reviewing@pf-skills --scope local    # just you, .claude/settings.local.json
```

Pick one delivery lane per project (`engineering` or `one-shot`), co-install `building`, and run `/setup-pf-skills` once in the repo.

To pick up catalog changes:

```bash
claude plugin marketplace update pf-skills
claude plugin update <plugin>@pf-skills
```

## Known overlaps

- `one-shot:code-review` ↔ `engineering:code-review` — complementary jobs (parallel-lens findings packet feeding the one-shot cascade vs two-axis Standards/Spec review), but both trigger on "review this branch"; install whichever matches the lane the project uses.
- `reviewing:review-branch` ↔ both `code-review` skills — `review-branch` is the deep human-facing pre-merge review; the code-review skills are lane machinery. Co-installing is fine — invoke the one you mean by name. Claude Code's built-in `/code-review` also answers this trigger.
- `one-shot:one-shot` ↔ `one-shot:aoe-orchestrate-plan` — split by trigger phrase: "run the pipeline" / "ship this" is the single-session cascade; "orchestrate the plan" / "work the tagged stack" is the aoe-driven stacked-PR flow. `aoe-orchestrate-plan` is slash-only, since it spawns billed sessions, and requires the external `aoe` CLI.

## Credits

The `engineering` and `productivity` plugins are largely vendored from [mattpocock/skills](https://github.com/mattpocock/skills) (MIT), several with local modifications. `marketing:humanizer` is based on [blader/humanizer](https://github.com/blader/humanizer) (MIT).

## License

[MIT](LICENSE).
