# Tokens: `@theme` → StyleX vars (P2)

Turn the project's Tailwind v4 theme into StyleX's token system. The result is a single source of truth that StyleX components, Astro scoped styles, and escape-hatch CSS all consume.

## Read the sources

Tailwind v4 theme lives in CSS, not JS config. Collect, in order:

1. `@theme` and `@theme inline` blocks — namespaced tokens: `--color-*`, `--font-*`, `--spacing-*`, `--breakpoint-*`, `--radius-*`, `--shadow-*`, `--animate-*`, plus one-offs like `--default-transition-duration`.
2. Hand-written variable blocks the theme aliases (`:root { --background: … }` with a `.dark { … }` override is the common shape).
3. `@custom-variant` definitions (usually the dark-mode variant — tells you the dark strategy).
4. Used-≥2-times literal values that never got a token (found in P0 census) — they earn one now only if `@theme` implies a scale for them; otherwise they wait for P5.

## Where each token goes

- **`defineVars`** — anything themeable or referenced by escape-hatch CSS: colors, font stacks, radii, semantic spacing.
- **`defineConsts`** — static values needing no runtime variable: breakpoint media-query strings, z-index layers, durations/easings. Consts inline at build time and cannot be themed.

File rules (compiler-enforced): `defineVars`/`defineConsts` must live in `*.stylex.ts` files, as named exports, flat objects, with nothing else exported from the file. Import these files **directly** everywhere — barrel re-exports break StyleX's static analysis.

```ts
// src/styles/tokens.stylex.ts
import * as stylex from '@stylexjs/stylex';

export const colors = stylex.defineVars({
  '--background': 'light-dark(oklch(98% 0.01 95), oklch(22% 0.02 270))',
  '--foreground': 'light-dark(oklch(25% 0.02 270), oklch(95% 0.01 95))',
});

export const fonts = stylex.defineVars({
  '--font-sans': '"Manrope", ui-sans-serif, system-ui, sans-serif',
});
```

```ts
// src/styles/consts.stylex.ts
import * as stylex from '@stylexjs/stylex';

export const breakpoints = stylex.defineConsts({
  sm: '@media (width >= 40rem)',   // copy the exact queries Tailwind emitted
  md: '@media (width >= 48rem)',
  maxMd: '@media (width < 48rem)',
});
```

## The bridge: literal `--` keys

A `--`-prefixed key makes `defineVars` emit that literal CSS variable name instead of a hashed one. That is the bridge: StyleX code references `colors['--background']`, and escape-hatch CSS / Astro scoped styles reference `var(--background)` — one name, one definition.

**Collision rule**: when reusing the project's existing variable names, delete the old `:root`/`.dark` definitions in the same commit that lands the tokens, and let the diff-clean gate prove nothing shifted. Two live definitions of one variable during coexistence makes dark mode ambiguous.

While Tailwind coexists (P2–P5), keep `@theme` working by aliasing it to the bridge: `@theme inline { --color-background: var(--background); }` — Tailwind utilities and StyleX then resolve to the same values, which is what keeps mid-migration screenshots diff-clean.

## Dark mode

Pick by who controls `color-scheme`:

**`light-dark()` (default recommendation).** When the project toggles a class/attribute that can set `color-scheme` — including three-way system/light/dark toggles — encode both palettes in one token as above, and reduce the old dark-mode machinery to:

```css
:root { color-scheme: light dark; }   /* system default */
.light { color-scheme: light; }
.dark  { color-scheme: dark; }
```

The toggle JS keeps flipping the class; `light-dark()` resolves per element. The entire `.dark { … }` override block and every `dark:` utility collapse into the token file. Baseline browser support since 2024; color values only.

**`createTheme` (alternative).** When non-color tokens differ per theme, or the project can't route through `color-scheme`: define light values as `defineVars` defaults, create the dark theme, and have the toggle apply the theme's generated class to the root:

```ts
// src/styles/theme-dark.stylex.ts
import * as stylex from '@stylexjs/stylex';
import { colors } from './tokens.stylex';

export const darkTheme = stylex.createTheme(colors, { '--background': 'oklch(22% 0.02 270)' });
// toggle JS: document.documentElement.className = dark ? stylex.props(darkTheme).className : '';
```

Overrides are partial — unlisted vars keep their defaults. Requires `unstable_moduleResolution` in the babel config.

**Media-strategy projects** (`dark:` maps to `prefers-color-scheme`, no toggle): either `light-dark()` with no class rules, or conditional token values:

```ts
'--background': { default: '#fff', '@media (prefers-color-scheme: dark)': '#111' },
```

Whichever route: after P2, `dark:` utilities in components translate to *nothing* — the property just references the token, and the token carries both modes. A `dark:` class that changes something with no token (e.g. `dark:hidden` icon swaps) becomes a condition key on the element only if the project controls a selectable state; otherwise give it a token (e.g. `--display-when-dark`) and note it.

## Derived tokens

Tokens may derive from siblings via arrow functions — emitted as CSS var references, so they track theme changes:

```ts
export const colors = stylex.defineVars({
  '--ink': 'light-dark(black, white)',
  '--ink-soft': () => `color-mix(in oklab, ${colors['--ink']} 70%, transparent)`,
});
```
