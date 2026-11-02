# Solution Format

Solution docs live in `docs/solutions/` as `<slug>.md`. Flat directory — the track lives in frontmatter, not in subfolders.

Create the `docs/solutions/` directory lazily — only when the first solution is written.

## Frontmatter

```md
---
title: <one line — the problem, not the fix>
date: YYYY-MM-DD
track: bug | knowledge
tags: [phoenix, auth, docker]
links: []   # related docs/adr/*, CONTEXT.md, or other docs/solutions/* paths
---
```

Keep it small. No enums to memorise, no validator. `track` and `date` are the only required fields beyond `title`.

## Body — `bug` track

```md
## Symptom
What was observed. The error message, quoted exactly, if there was one.

## What failed
The approach(es) tried that did not work, and why they looked plausible.

## Fix
What actually resolved it. The diff in words, or the key line.

## Root cause
The real underlying reason — the thing that, if understood earlier, would have skipped the whole detour.

## Prevention
The check, test, or convention that stops this recurring.
```

## Body — `knowledge` track

```md
## Context
The situation where this knowledge applies.

## Guidance
The actual thing to know or do.

## Why it matters
The cost of not knowing it.

## When to apply
The trigger conditions. When this is relevant — and when it is not.

## Example
A concrete instance, ideally from this repo.
```

## Principles

- The `title` names the **problem**, not the solution — that is what a future searcher types.
- Drop any section that would be empty. A two-line solution doc is fine.
- Tag inherited auto-memory evidence inline as `(prior learning)`.
- Cross-link liberally in `links:` — to the ADR that explains the decision behind the code, to the CONTEXT.md term, to sibling solutions.
- A solution doc is a postmortem or a how-it-works note. It is **not** a glossary entry (that is `CONTEXT.md`) and **not** a decision record (that is `docs/adr/`).
