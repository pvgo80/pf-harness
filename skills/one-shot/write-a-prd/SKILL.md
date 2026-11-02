---
name: write-a-prd
description: >
  Create a product requirements document through user interview, codebase
  exploration, and module design. Use when the user wants to write a PRD, plan a
  new feature, or create a product spec.
metadata:
  version: 0.2.0
---

# Write a PRD

Create a comprehensive product requirements document by interviewing the user
and exploring the codebase.

## Process

### Phase 0: Determine PRD location

Before anything else, determine where to save the PRD. Check `docs/agents/prds.md`:

- If it exists, read the line starting with `**Location:**` and use that path
- If it does not exist, default to `prds/`

The chosen path is referred to as `<prd-location>` below.

### Phase 1: Interview

1. Ask what the user wants to build and why.
2. Clarify scope — what's in, what's explicitly out.
3. Identify users/personas affected.
4. Ask about constraints: timeline, tech stack, dependencies, compliance.
5. Ask about success criteria — how will we know this worked?

If the current conversation already answers a question, synthesize the answer
and confirm it rather than asking from scratch. Only interview for what's
genuinely missing.

### Phase 2: Codebase exploration

1. Explore the current codebase to understand existing architecture.
2. Identify where the new feature fits in.
3. Note existing patterns, conventions, and abstractions to build on.
4. Flag potential conflicts or areas that need refactoring.
5. Read the domain docs and apply them to this feature:
   - `CONTEXT.md` (or the relevant context's `CONTEXT.md` via `CONTEXT-MAP.md`)
     — reuse the established ubiquitous language; don't introduce synonyms for
     terms that already exist. Flag any new term the feature needs.
   - `docs/adr/` — check for decisions that constrain this feature. Note any
     the design must respect, or must explicitly supersede.

### Phase 2.5: Testing seam

Decide where this feature will be tested before designing it — the seam shapes
the design.

1. Identify the integration seam where the feature's behaviour is exercised.
   Prefer an existing seam over a new one, at the highest architectural level
   that still gives meaningful coverage.
2. Confirm the seam with the user before writing the PRD.
3. Record it under **Testing** in the PRD (see below).

### Phase 3: Write the PRD

Output a document in `<prd-location>` (from Phase 0) with the following structure:

```markdown
# [Feature Name]

## Problem
What problem does this solve and for whom?

## Goals
Numbered list of concrete, measurable goals.

## Non-goals
What this feature explicitly does NOT do.

## Design

### User experience
How the user interacts with this feature.

### Technical design
Architecture, data model, API changes, key modules.

### Dependencies
External services, libraries, or other features required.

## Implementation plan
High-level phases or milestones.

## Testing
The integration seam(s) where this feature is exercised, and the external
behaviour each verifies.

## Success criteria
How we measure whether this feature achieved its goals.

## Open questions
Anything unresolved that needs further discussion.
```

### Phase 4: Review

Walk the user through the PRD section by section. Incorporate feedback until
they approve it.

## Guidelines

- Keep the PRD focused and actionable — no fluff.
- Reference specific files, modules, and functions from the codebase.
- If the codebase exploration reveals that the original idea needs adjustment,
  say so clearly.
- Save the final PRD to `<prd-location>/<feature-name>.md`.
