---
name: build-lofi
description: Use when an IA exists and the project needs a navigable wireframe — multi-screen greyscale Astro prototype where buttons and links actually work, suitable for moderated user testing. Triggered by "build the lo-fi", "scaffold the wireframe", "make the prototype clickable", or "let's user-test the flow". Has subcommands `regen <screen>` and `debrief` for iterating after first build.
---

# Build a lo-fi

Generates a navigable greyscale Astro wireframe at `design/lofi/` from `docs/design/ia.md`. Wires `<a>` links per flow definitions, enforces no-color via a lint step, and supports two follow-up modes:
- `build-lofi regen <screen>` — regenerate one screen idempotently.
- `build-lofi debrief` — post-user-test interview, writes findings, offers inline regens.

## When to use

- IA is locked, ready for visual exploration.
- Need a clickable prototype for moderated user testing.
- Need to iterate quickly between IA edits and visual exploration.

When NOT to use:
- Ad-hoc design exploration without an IA — use the existing `prototype` skill.
- Hi-fi UI build — that's downstream from PRD + design system.

## Missing prerequisites

If `docs/design/ia.md` doesn't exist, warn:

> "No IA found at `docs/design/ia.md`. The lo-fi needs an IA to know which screens to build and how they link. Want to run `write-an-ia` first? [y / n / skip-and-provide-inline]"

Offer to chain into `write-an-ia`. Skip is allowed only if the user explicitly opts in.

## Subcommand routing

- Plain `build-lofi` → **default mode** (full generate)
- `build-lofi regen <screen> [with <instruction>]` → **regen mode**
- `build-lofi debrief` → **debrief mode**

## Default mode: generate

### 1. Read and validate IA

Parse `docs/design/ia.md`. Extract:
- Glossary terms
- Sitemap (screen name → purpose, with parent relationships)
- Flows (named journeys as arrow strings)

Validate: every flow screen exists in the sitemap with exact name match. If broken (user hand-edited the IA), report orphan references and stop.

### 2. Staleness check

If `design/lofi/.lofi-generated` exists and `docs/design/ia.md` has an mtime ≤ the sentinel's, the lo-fi is up-to-date. Offer:
- "Lo-fi appears current. Regenerate anyway? [y/n]"
- Default: n. Suggest `build-lofi regen <screen>` for surgical updates instead.

If the IA is newer than the sentinel, mark every screen as "may be stale", let the user pick which to regen (or all).

### 3. Flag ambiguities

Scan screens for vagueness:
- Purpose under 4 words ("main page", "user panel").
- Generic name without qualifier ("Dashboard", "Home", "Page", "Detail").
- Screen appears in 2+ flows with different roles.
- Flow branch point: same source goes to 2+ destinations across flows.

For each ambiguity, **explain why** before asking. Example:

> "Cart" appears in the sitemap with purpose "checkout step". Referenced in two flows — "Add to cart" ending at "Cart", and "Apply coupon" starting at "Cart". The lo-fi needs to know what's on this screen.
>
> (a) Tell me the content: e.g. "list of items, total at bottom, apply-coupon link, checkout CTA"
> (b) Push back to IA: split into two distinct screens.

If the user picks (b), tell them to update the IA and re-run.

### 4. Scaffold the Astro project (only if `design/lofi/` doesn't exist)

Copy from this skill's `templates/` directory:
- `package.json` — Astro + Tailwind v3
- `astro.config.mjs`
- `tailwind.config.mjs` — greyscale-only baseline
- `src/layouts/Layout.astro`
- `src/components/Header.astro`, `Footer.astro`
- `src/styles/tokens.css` — greyscale-only baseline (gets replaced when `design-system` materialises tokens)
- `lofi-lint.mjs` — enforces no color
- `JOURNEY.md` and `LOFI.md` placeholders

Add an npm `prebuild` script that runs `node lofi-lint.mjs`.

### 5. Generate per-screen pages

For each sitemap screen, write `src/pages/<slug>.astro`:
- Slug = kebab-case of screen name (`Meeting detail` → `meeting-detail`).
- Imports `Layout`, `Header`, `Footer`.
- `<h1>` = screen name. Purpose from IA as subtitle.
- Body: greyscale wireframe elements matching purpose (boxes for content, gray rectangles for images, lists with placeholder rows).
- Navigation: every flow containing this screen contributes a `→ next-screen` link. Multiple outgoing destinations → labeled list ("Continue to Cart" vs "Apply coupon").

Use mock data consistent with `JOURNEY.md`. If `JOURNEY.md` is empty, ask the user one paragraph about the assumed user state, then write it.

### 6. Write narrative documents

- `JOURNEY.md` — one paragraph: who is the user, what state are they in. Names, project counts, mock content scope. Every screen reads from this.
- `LOFI.md` — per-screen invented content summary (what mock data, what buttons, where each link goes).

### 7. Touch the sentinel

Write the current timestamp to `design/lofi/.lofi-generated`.

### 8. Lint

Run `node design/lofi/lofi-lint.mjs`. The lint scans `.astro` files and `tokens.css` for non-greyscale color utilities. Any hit → fail with file + line.

### 9. Report

Print:
- Path written
- Screen count
- Ambiguities resolved
- Astro dev command: `cd design/lofi && npm install && npm run dev`

## Regen mode: `build-lofi regen <screen> [with <instruction>]`

1. Parse screen name (kebab-case or sentence-case). Verify it exists in IA. If not, error.
2. Rewrite ONLY `src/pages/<slug>.astro`. Do not touch layouts, other screens, or styles.
3. If instruction provided, incorporate it; otherwise regenerate from current IA + JOURNEY.md.
4. Update that screen's entry in `LOFI.md`.
5. Touch the sentinel.
6. Run the lint.

## Debrief mode: `build-lofi debrief`

Post-user-test interview. Writes/appends `docs/design/lofi-notes.md`.

### 1. Confirm context

Ask one at a time:
- Date of session (default: today)
- Tester name or handle
- Which flow(s) were tested

### 2. Walk the template

For each section, ask, capture, move on. Don't dump all questions at once.

```markdown
## Flow validity
For each flow tested: did it work end-to-end? Where did the tester pause, backtrack, or quit?

## Per-screen friction
For each screen visited: what confused them, what they expected, what they did instead.

## IA changes proposed
Any screens that should be added, removed, renamed, or have their purpose rewritten.

## Regen requests
Any screen that should be regenerated, with the instruction (e.g. "Cart: simpler coupon UI").

## Open questions for next test
Things the designer wants to probe next time.
```

### 3. Write the notes

Append to or create `docs/design/lofi-notes.md`. Each session is its own `## Session <date> with <tester>` block.

### 4. Offer inline regens

Summarise the captured regen requests:

> "Captured 2 regen requests:
> 1. Cart — 'simpler coupon UI'
> 2. Onboarding — 'fewer steps'
>
> Run these now? [all / pick / skip]"

- "all" → chain into `build-lofi regen` for each.
- "pick" → numbered list, user selects.
- "skip" → notes preserved as TODOs in `lofi-notes.md`.

Then summarise proposed IA changes and offer: "Apply via `write-an-ia`? It'll open the editor with these notes in scope." Don't auto-edit IA.

## Common mistakes

- **Color slipping in.** No Tailwind color utilities except `bg-white`, `bg-black`, `bg-gray-*`, `text-gray-*`, `border-gray-*`. Lint catches it; respect the failure.
- **Adding state.** No `useState`, no fetches, no form submissions. Buttons that "submit" navigate via `<a href>`.
- **Mock data drift.** Anna's name appears the same way on every screen. Three projects everywhere = the same three projects. `JOURNEY.md` is the source.
- **Skipping ambiguity flagging.** Generic screen names produce vague wireframes. Always flag and ask.
- **Touching multiple files in regen mode.** Regen Cart → touch only `cart.astro` and `LOFI.md`. Layout changes warrant a full regenerate.
- **Forgetting the sentinel.** Every successful generate/regen must touch `.lofi-generated`. Staleness detection depends on it.

## Red flags — STOP

- IA validates as broken (orphan flow references). Refuse and report; user fixes IA first.
- More than 3 ambiguous screens flagged. Push back: IA needs sharpening.
- User asks for color in lo-fi. Decline. Greyscale enforces flow validity testing without aesthetic distraction.
- Debrief mode without any session having happened. Ask: "Was there a session, or are you starting one?" Don't fabricate notes.
- Lint fails and user asks you to bypass it. Don't. Fix the color leakage instead.
