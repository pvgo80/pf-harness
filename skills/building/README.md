# Building

The shared foundation both delivery lanes start from: sharpen an idea by
interview, answer hard design questions with throwaway prototypes, hand work
between sessions, and capture what was learned. The lanes themselves live
elsewhere — `engineering` (spec → tickets → implement, tracker-based) and
`one-shot` (orchestrated PRD lane) — and both are documented co-installs of
this plugin.

## Skills

- **[grill-me](./grill-me/SKILL.md)** — Stress-test a plan or design through relentless interviewing. Walks each branch of the decision tree.
- **[grill-with-docs](./grill-with-docs/SKILL.md)** — Grilling session that cross-references `CONTEXT.md` and ADRs, sharpens terminology, and updates documentation inline as decisions crystallise.
- **[prototype](./prototype/SKILL.md)** — Build a throwaway prototype to flesh out a design — a runnable terminal app for state/business-logic questions, a shareable HTML demo for stakeholders, or several radically different UI variations toggleable from one route.
- **[handoff](./handoff/SKILL.md)** — Compact the current conversation into a handoff document so another agent can continue the work.
- **[compound](./compound/SKILL.md)** — Capture a just-solved problem into `docs/solutions/` and promote cross-product patterns to auto-memory. The back half of the loop.
- **[zoom-out](./zoom-out/SKILL.md)** — Tell the agent to zoom out and give broader context or a higher-level perspective on an unfamiliar section of code.

## How it's used

```
grill-me / grill-with-docs   Sharpen the idea (docs variant when a repo is under you)
        |
   [prototype]               Detour when a question needs a runnable answer,
        |                    bridged by handoff in both directions
        v
  a delivery lane            engineering (to-spec → to-tickets → implement)
                             or one-shot (write-a-prd → prd-to-issues → workers)
        |
    compound                 Capture what was learned after a fix or a swarm
```

Run `setup-pf-skills` (in the `setup` plugin) once per repo before the first
grilling session so domain docs land where the other plugins expect them.
