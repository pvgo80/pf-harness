---
name: prd-to-issues
description: >
  Break a PRD into independently-grabbable issues using tracer-bullet vertical
  slices, published to the issue tracker or a local issues file per the repo's
  issues mode. Use when the user wants to convert a PRD to issues, create
  implementation tickets, or break down a spec into work items.
metadata:
  version: 0.2.0
---

# PRD to Issues

Break a product requirements document into independently-grabbable issues that
can be worked on in sequence or parallel.

## Process

0. **Determine PRD location.** Check `docs/agents/prds.md`. If it exists, read the line starting with `**Location:**` and use that path. If not, default to `prds/`. Call this `<prd-location>`.
   **Resolve the issues mode.** Read the `**Issues:**` line in `docs/agents/prds.md`: `tracker` or `file`. With no such line, it's `tracker` if `docs/agents/prds.md` says in prose that issues live on the tracker (a legacy override) or `docs/agents/issue-tracker.md` names GitHub or GitLab, and `file` otherwise. In `tracker` mode, run every issue operation the way `docs/agents/issue-tracker.md` describes (GitHub via `gh` when that file is absent); no local `*-issues.md` file exists.
1. Read the PRD. If multiple PRDs exist in `<prd-location>`, ask the user which one.
2. Identify the **tracer bullet** — the thinnest vertical slice that touches
   every layer (UI, API, database, etc.) and proves the architecture works.
3. Break the remaining work into issues that:
   - Are independently implementable (no circular dependencies).
   - Have clear acceptance criteria.
   - Are small enough to be completed in a single session.
   - Follow a logical dependency order.

## Issue format

Each issue should contain:

```markdown
## [Issue title]

**Depends on:** [list of issue titles this depends on, or "none"]

### Description
What needs to be built and why.

### Acceptance criteria
- [ ] Concrete, testable criteria
- [ ] Another criterion

### Technical notes
Relevant files, modules, patterns to follow, edge cases.
```

## Refactor-shaped work

Some work isn't a vertical slice — it's a wide mechanical change (rename,
schema migration, API version bump, library swap) that touches many call sites.
Don't force it into a tracer bullet. Sequence it as **expand → contract**,
keeping CI green at every step:

1. **Expand** — add the new form alongside the old one, without removing the old.
   One issue.
2. **Migrate** — move call sites over in blast-radius batches. One issue per
   batch, each independently mergeable.
3. **Contract** — delete the old form once nothing references it. One issue,
   depends on all migrate batches.

Use this only for genuinely wide, mechanical changes. Feature work stays a
tracer-bullet vertical slice.

## Output

Draft all issues in dependency order, the tracer bullet first. Then publish
them per the issues mode.

**`file` mode.** Save the issues to `<prd-location>/<feature-name>-issues.md` as
a single document.

**`tracker` mode.** Create one tracker issue per slice, in dependency order, so
issue numbers ascend with it:

- Title: the issue title. Body: the issue format above without the
  `## [Issue title]` heading and the `**Depends on:**` line, plus a first line
  `PRD: <prd-location>/<feature-name>.md`.
- Label every issue `prd:<feature-name>`, creating the label first if it
  doesn't exist.
- Set each dependency as a blocking edge, per the **Blocking** entry in
  `docs/agents/issue-tracker.md`: the native link, or the `Blocked by: #<n>`
  line where native links aren't available. Create blockers before the issues
  they block, so every edge points at an existing number.
- Put the dependency graph summary and the parallel-work flags in your reply,
  with each issue's number and URL. Write no local file.

## Guidelines

- Prefer vertical slices (feature through all layers) over horizontal slices
  (one layer at a time).
- Each issue should deliver visible, testable progress.
- Include a dependency graph summary at the top of the document.
- Flag any issues that could be worked in parallel.
- Keep issues small — if an issue has more than 5 acceptance criteria, consider
  splitting it.
