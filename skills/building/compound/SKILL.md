---
name: compound
description: >
  Capture a just-solved problem (a bug fix or a piece of hard-won knowledge) into
  the repo's searchable knowledge base, and promote cross-product patterns to
  auto-memory. The back half of the build loop. Use after a fix lands, after an
  issue-swarm finishes, after a debugging session, or when the user says
  "capture this", "document what we learned", "compound this", or runs /compound.
metadata:
  version: 0.4.0
---

<what-to-do>

Document a recently-solved problem so the next person (or agent) finds it instead of re-solving it. You are the coordinator. Research happens in parallel subagents you dispatch; only you write files.

Run these phases in order:

0. **Scan auto-memory** for related prior learnings.
1. **Dispatch 3 research subagents in parallel.**
2. **Assemble and write** one solution doc (or update an existing one).
3. **Route by scope** — promote cross-product patterns to auto-memory.
4. **Report** where it landed.

If the user gave no context hint, first auto-scan the recent session and any touched issues for capture-worthy moments (a bug that was diagnosed and fixed, a non-obvious decision, a surprising gotcha). Surface the candidates as a short list and let the user pick which to capture. One run captures one learning.

</what-to-do>

<supporting-info>

## Parameters

- Context hint — optional free text after `/compound` ("the colocated-hooks Dockerfile fix"). Narrows the scan.
- Store location — `docs/solutions/`. Flat directory; `track` lives in frontmatter, not in subfolders.

## Phase 0 — Scan auto-memory

Check the system prompt for auto-memory entries (Claude Code surfaces these in a `<system-reminder>`). If any relate to the problem being captured, pass their content to the research subagents as supplementary evidence, and note them as promotion candidates for Phase 3. If there is no auto-memory, skip silently.

## Phase 1 — Dispatch 3 research subagents in parallel

Send all three in a single tool-call batch. Each returns **text only** — subagents never write files. Each prompt is self-contained; do not pass raw session history, pass a distilled description of the problem and its resolution.

**Context analyzer** — classify and locate.
- Decide the track: `bug` (something broke and was fixed) or `knowledge` (how something works / a reusable approach).
- Propose a kebab-case slug for the filename.
- Return: track, slug, one-line title, suggested tags.

**Solution extractor** — draft the body.
- `bug` track: Symptom · What failed · Fix · Root cause · Prevention.
- `knowledge` track: Context · Guidance · Why it matters · When to apply · Example.
- Incorporate any auto-memory evidence from Phase 0, tagged inline as `(prior learning)`.
- Return: a filled-in body following [SOLUTION-FORMAT.md](./SOLUTION-FORMAT.md).

**Overlap finder** — prevent duplicates.
- Grep `docs/solutions/`, `CONTEXT.md`, and `docs/adr/` for the same problem area.
- Score overlap: **high** (4-5 matching dimensions — same component, same symptom, same fix), **moderate** (2-3), **low/none** (0-1).
- Return: overlap score, the path of any high-overlap doc, related docs to cross-link, and any stale doc that this learning should refresh.

## Phase 2 — Assemble and write

Wait for all three subagents. Then:

1. If the overlap finder reports **high** overlap → update the existing doc in place (merge the new detail, refresh the date). Do not create a duplicate.
2. Otherwise → create `docs/solutions/<slug>.md`. Run `mkdir -p docs/solutions/` first; create the directory lazily — only now, when there is something to write.
3. Use the lean frontmatter + body from [SOLUTION-FORMAT.md](./SOLUTION-FORMAT.md). Fill `links:` with related ADR / CONTEXT / solution paths the overlap finder found.
4. Never put glossary terms in a solution doc — those belong in `CONTEXT.md`. Never restate an architecture decision — that is an ADR. A solution doc is a postmortem or a how-it-works note.

**You are the only writer.** Subagents returned text; you assemble and write the single file.

## Phase 3 — Route by scope (automatic)

Decide whether the learning is repo-local or cross-product, and act without prompting:

- **Repo-specific** (tied to this codebase's code, schema, or domain) → the `docs/solutions/` doc is the whole output. Stop.
- **Cross-product** (a pattern, gotcha, or convention that will recur in other products / other repos) → *also* write it to auto-memory:
  - Write a memory file under the user's auto-memory directory with the appropriate `type` (`reference` for patterns, `feedback` for workflow guidance, `project` for ongoing constraints).
  - The memory body must reference the solution doc by path, so the cross-product note points back to the full per-repo detail.
  - Add the one-line pointer to `MEMORY.md`.
  - Before writing, check for an existing memory that already covers it and update that instead of duplicating.

The scope test in one line: *would this help me on a different product?* Yes → memory too. No → doc only.

## Phase 4 — Report

Print:
- The doc path written or updated, and whether it was a create or an update.
- Whether a memory was also written (and its name), or "repo-local, no memory".
- Any stale doc flagged for refresh (mention it; do not auto-edit unrelated docs).

## Discoverability (first run only)

If `docs/agents/` exists but has no `solutions.md`, offer to drop a consumption guide so other agents know the store exists and how to read it. Use the template in [SOLUTIONS-GUIDE.md](./SOLUTIONS-GUIDE.md). Ask before writing it. Skip silently on later runs.

## Constraints

- Coordinator only writes files. All research goes through the 3 parallel subagents.
- One learning per run. One doc (plus an optional memory pointer).
- Create `docs/solutions/` lazily — never scaffold an empty tree.
- High overlap → update in place, never duplicate.
- Solution docs hold postmortems and how-it-works notes — not glossary terms (CONTEXT.md) and not decisions (docs/adr/).
- Do not edit ADRs or `CONTEXT.md` from this skill.

## Failure modes to watch

- Nothing capture-worthy in the session → say so and stop. Do not invent a learning.
- Overlap finder returns high overlap but the existing doc is wrong → update it, note the correction in Phase 4.
- No auto-memory directory available (non-Claude-Code host) → Phase 3 writes the doc only and reports that memory promotion was unavailable.

</supporting-info>
