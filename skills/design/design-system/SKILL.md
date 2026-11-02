---
name: design-system
description: Use when establishing the visual foundation for a new project — typography scale, color palette, spacing — as canonical design tokens. Triggered by "set up the design system", "pick fonts and colors", "define design tokens", or when an IA exists but no docs/design/system/ files do.
---

# Design system

Two-pass interactive flow: typography first, then color. Outputs canonical design tokens as TypeScript constants, materialised into a CSS-variable block and a Tailwind config.

This skill anchors the project's visual language. Typography drives the spacing scale; the palette is generated from a vibe interview (or a supplied brand seed) into perceptually uniform OKLCH tint scales. Light mode only in v1.

## When to use

- Typography and palette aren't yet defined for a project.
- The lo-fi needs runtime-swappable variations (`?scale=`, `?palette=`) for design comparison.
- Moving from IA → visual definition.

When NOT to use:
- Editing an existing token — edit the source file directly.
- Picking a *brand identity* — this skill produces a design system around an existing brand direction, not a brand from scratch.

## Missing prerequisites

If `docs/design/ia.md` does not exist, the design system can still be written, but the lo-fi (which consumes these tokens) requires an IA. Warn the user. Offer to run `write-an-ia` first. Allow skip.

## Output locations

Source of truth:
- `docs/design/system/scale.ts` — typography + spacing tokens (TypeScript constants)
- `docs/design/system/palette.ts` — color tokens (TypeScript constants)

Materialised for the lo-fi (if `design/lofi/` exists):
- `design/lofi/src/styles/tokens.css` — `:root { --... }` block
- `design/lofi/tailwind.config.mjs` — references `var(--...)`

If multiple variants are generated for comparison:
- `design/lofi/scale-variants/{name}.css` — alternate scale token blocks
- `design/lofi/palette-variants/{name}.css` — alternate palette blocks

If `design/lofi/` doesn't exist yet, write only the source-of-truth files and tell the user the lo-fi will pick them up next time `build-lofi` runs.

## Process

### Pass 1: Typography

1. **Specimen category.** Ask: sans-serif (default), serif, monospace, or display? Don't ask for a specific font yet — the `fontFamily` block gets a `TODO: project font` placeholder; system fallback stack is filled per category.

2. **Pick presets.** Show the four curated presets plus an optional custom tuple:

   | Preset | Base | Ratio | Character |
   |---|---|---|---|
   | Compact | 16 | 1.2 | fast, dense |
   | Balanced | 16 | 1.333 | steady |
   | Relaxed | 18 | 1.414 | calm |
   | Spacious | 20 | 1.5 | airy |
   | Custom | user-supplied (base, ratio) | — |

   Ask the user to pick one preset, or "show me a few to compare." Multiple → emit as variants under `scale-variants/`.

3. **Compute the scale.** Invoke `node generate-scale.mjs typography <base> <ratio>` (see `generate-scale.mjs` in this skill directory). The script returns:
   - `sizes`: { caption, body, h3, h2, h1 } — each snapped to a multiple of 4, with anti-collision applied.
   - `lineHeights`: matching, each a multiple of 8 (baseline grid is half, so a multiple of 4).
   - `spacing`: { xs:4, s:8, m:16, l:24, xl:32, xxl:48 } — fixed for v1.
   - `warnings`: any string describing collisions that required ≥2 bumps.

4. **Surface warnings.** If `warnings` is non-empty, print the message verbatim. Soft warning only — do not block, do not auto-pick another preset. Let the user decide.

### Pass 2: Color

1. **Brand seed check.** Ask: "Do you already have brand colour(s) you need to lock — hex, hue, or a brand name?" If yes → skip the vibe interview; convert the supplied input into a seed hue/chroma pair and continue.

2. **Vibe interview** (only if no seed). Five questions, in order:
   1. What domain or industry is the product in?
   2. Mood on three axes: warm↔cool, calm↔energetic, sophisticated↔playful.
   3. Reference brand to lean toward + one to feel distinct from.
   4. Audience sophistication.
   5. Anything to actively avoid.

3. **Propose seed hues** (LLM reasoning from the interview). Surface these to the user before generating; let them tweak.
   - Brand hue (H 0–360°), chroma 0.10–0.20 (low for sophisticated/calm, high for energetic/playful).
   - Neutral hue, chroma 0.01–0.03, biased ±15° from brand for tonal harmony.
   - Optional accent hue (90° or 180° from brand) if the vibe needs a counter-colour.
   - Semantic placeholders: success 145°, warning 70°, error 25°, info 240°. These are fixed per-language convention; only the lightness curve adapts.

4. **Generate tint scales.** Invoke `node generate-scale.mjs color <kind> <hue> <chroma>` for each scale (brand, neutral, accent if any, four semantics). The script returns 10 steps (50, 100, …, 900) as `{ oklch, hex }` pairs.

5. **WCAG nudge loop** (max 2 iterations).
   - Check contrast: `brand-500` on `neutral-50`, and `neutral-50` on `brand-500`.
   - If either is below 4.5:1, the script returns a nudged scale with the L curve adjusted down 0.02 at the relevant steps.
   - After 2 nudges, if still failing, surface the failure. Don't ship a low-contrast palette silently.

6. **Generate variants if requested.** "Show me 3 palettes" → generate three with slightly varied hue/chroma seeds, write to `palette-variants/`.

## Anti-collision rule (typography)

Body is the anchor. Snap each scale level to nearest multiple of 4. Walk levels above body upward: each must be ≥ previous + 4 (bump if not). Walk levels below body downward: each must be ≤ previous − 4 (bump if not).

If any level required ≥2 bumps, surface a soft warning: "Ratio X at base Y collides too tight. Try a wider ratio or larger base." Continue writing — do not hard-stop. The user picks.

## Token format

### `docs/design/system/scale.ts`
```typescript
export const scale = {
  fontFamily: {
    // TODO: project font
    sans: 'ui-sans-serif, system-ui, -apple-system, sans-serif',
  },
  base: 16,
  ratio: 1.333,
  sizes: { caption: 12, body: 16, h3: 20, h2: 24, h1: 32 },
  lineHeights: { caption: 16, body: 24, h3: 28, h2: 32, h1: 40 },
  spacing: { xs: 4, s: 8, m: 16, l: 24, xl: 32, xxl: 48 },
} as const;
```

### `docs/design/system/palette.ts`
```typescript
export const palette = {
  brand: {
    50: { oklch: 'oklch(0.985 0.02 240)', hex: '#f6faff' },
    // ... 100–900
  },
  neutral: { /* same shape */ },
  semantic: {
    success: { default: { oklch: '...', hex: '...' }, bg: { oklch: '...', hex: '...' } },
    warning: { /* ... */ },
    error:   { /* ... */ },
    info:    { /* ... */ },
  },
} as const;
```

### `design/lofi/src/styles/tokens.css`
```css
:root {
  /* Typography */
  --font-sans: ui-sans-serif, system-ui, -apple-system, sans-serif;
  --size-body: 16px;
  --line-body: 24px;
  /* ...other sizes/line-heights/spacing */

  /* Brand scale (OKLCH triplets — space-separated, no commas) */
  --brand-50: 0.985 0.02 240;
  --brand-100: 0.965 0.04 240;
  /* ...200–900 */

  /* Neutral, semantic — same pattern */
}
```

### `design/lofi/tailwind.config.mjs`
```js
import { scale } from '../../docs/design/system/scale.ts';

export default {
  content: ['./src/**/*.{astro,html,js,ts}'],
  theme: {
    extend: {
      fontFamily: { sans: 'var(--font-sans)' },
      fontSize: {
        caption: ['var(--size-caption)', { lineHeight: 'var(--line-caption)' }],
        body:    ['var(--size-body)',    { lineHeight: 'var(--line-body)' }],
        h3:      ['var(--size-h3)',      { lineHeight: 'var(--line-h3)' }],
        h2:      ['var(--size-h2)',      { lineHeight: 'var(--line-h2)' }],
        h1:      ['var(--size-h1)',      { lineHeight: 'var(--line-h1)' }],
      },
      spacing: scale.spacing,
      colors: {
        brand: {
          50:  'oklch(var(--brand-50))',
          100: 'oklch(var(--brand-100))',
          // ...
        },
        neutral: { /* ... */ },
        success: 'oklch(var(--success))',
        warning: 'oklch(var(--warning))',
        error:   'oklch(var(--error))',
        info:    'oklch(var(--info))',
      },
    },
  },
};
```

## Common mistakes

- **Hard-coding hex without OKLCH source.** Always store OKLCH primary, hex fallback. Editing hex by hand drifts the scale.
- **Forgetting the WCAG nudge.** Generated palettes can fail AA on key pairings. Run the check.
- **Letting the user pick "any font".** Push for the specimen category. Even "system sans-serif" is more useful than "TBD".
- **Generating both passes silently.** This is interactive. Ask, confirm, then write.
- **Skipping the brand-seed branch.** Most real projects have at least a brand color already. Always offer the bypass before the vibe interview.

## Red flags — STOP and ask

- User says "skip the vibe interview, just pick colors" without supplying a seed. Decline; ask for at least domain + mood.
- Generated palette fails WCAG after 2 nudges. Surface the failure; don't ship low-contrast tokens.
- Custom typography tuple produces ≥2 collision bumps on any level. Warn explicitly; offer to fall back to a curated preset.
- No `design/lofi/` exists and user wants variants. Only canonical files can be written without the lo-fi; explain.
