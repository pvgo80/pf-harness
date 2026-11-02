# Design

Skills for going from a brief to a navigable, user-testable wireframe.

## Skills

- **[write-an-ia](./write-an-ia/SKILL.md)** — Map the Information Architecture into `docs/design/ia.md`: glossary, sitemap, named user flows, and open questions. Canonical input to the rest of the workflow.
- **[design-system](./design-system/SKILL.md)** — Establish the visual foundation — typography scale, OKLCH color palette, spacing — as canonical design tokens, materialised into CSS variables and a Tailwind config.
- **[build-lofi](./build-lofi/SKILL.md)** — Generate a navigable greyscale Astro wireframe from the IA, with working links per flow, a no-color lint step, and `regen <screen>` / `debrief` subcommands for iterating.

## Workflow

```
 1. Write an IA       Map screens, flows, and glossary
       |
 2. Design System     Define typography, palette, tokens
       |
 3. Build Lo-fi       Scaffold a clickable greyscale prototype
```

You can enter at any step, but each builds on the one before: `build-lofi` reads the IA, and the lo-fi consumes the design tokens.
