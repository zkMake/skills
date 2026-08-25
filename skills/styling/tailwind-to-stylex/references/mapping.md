# Tailwind → StyleX mapping

How a resolved Tailwind class becomes StyleX. Load at the start of P4; return whenever a class carries a variant or resists a one-property translation. Authoring rules (camelCase, longhands, `default`, merge order) live in `gotchas.md` — this file covers where each Tailwind construct *goes*.

## Resolve, then reshape

Never translate a class name from memory. Find its actual declarations in `migration/baseline.css`:

```bash
# class selectors are escaped in compiled CSS: hover:bg-card/70 → .hover\:bg-card\/70
grep -A6 'hover\\:bg-card\\/70' migration/baseline.css
```

Copy values **verbatim** — v4 emits `oklch(...)` colors, `color-mix(...)` for opacity modifiers (`bg-card/70`), and `rem`-based media queries. Reproducing those exactly is what makes the pixel diff pass. If a class is absent from `baseline.css` (dynamically composed and purged, or from an unfamiliar plugin), render it once and read `getComputedStyle`, then flag it.

## Variants → condition keys

A variant becomes a key inside the property's value object, next to a mandatory `default` (use `default: null` when there is no base value):

```ts
// bg-card hover:bg-card/70 focus-visible:bg-card/50
backgroundColor: {
  default: 'var(--card)',
  ':hover': 'color-mix(in oklab, var(--card) 70%, transparent)',
  ':focus-visible': 'color-mix(in oklab, var(--card) 50%, transparent)',
},
```

| Tailwind | Condition key |
| --- | --- |
| `hover:` `focus:` `active:` `visited:` `disabled:` `checked:` `focus-visible:` `focus-within:` | `':hover'` etc. — same pseudo-class |
| `first:` `last:` `odd:` `even:` `empty:` | `':first-child'`, `':last-child'`, `':nth-child(odd)'`, `':nth-child(even)'`, `':empty'` |
| `not-hover:` and other `not-*` (v4) | `':not(:hover)'` etc. |
| `aria-expanded:` / `aria-[...]` | `'[aria-expanded="true"]'` — attribute-selector key |
| `data-[state=open]:` / `data-[open=true]:` | `'[data-state="open"]'` / `'[data-open="true"]'` |
| `has-[...]:` (self-targeting) | `':has(...)'` — note limited browser support; keep only if the project already shipped it |
| `print:` | `'@media print'` |
| `sm:` `md:` … and `max-*:` (v4) | `@media` key — copy the exact query from `baseline.css` (v4 uses `rem`/range syntax); centralize via `defineConsts` per `tokens.md` |
| `dark:` | a token concern, not a per-property one — see `tokens.md` |
| `starting:` (v4 `@starting-style`) | unverified in StyleX — test-compile once; if rejected, keep that rule in a scoped/escape-hatch stylesheet and flag it |
| Arbitrary variant targeting the element itself, e.g. `[&:nth-child(3)]:` | the selector as a condition key, e.g. `':nth-child(3)'` |
| Arbitrary variant reaching other elements, e.g. `[&>svg]:` | no direct home — restructure (below) |

Stacked variants nest, and **every nesting level needs its own `default`** — a missing inner default silently drops the style:

```ts
// hover:opacity-80 md:hover:opacity-60
opacity: {
  default: 1,
  ':hover': { default: 0.8, [consts.md]: 0.6 },
},
```

Tailwind's mobile-first stacking (`w-full md:w-1/2 lg:w-1/3`) flattens into one value object: unprefixed → `default`, each breakpoint → its media key. StyleX conditions are a discrete value switch per property, not a cascade — list every breakpoint's value explicitly.

## Values

- **Arbitrary values** read directly off the class: `top-[117px]` → `top: '117px'`; `text-[11px]` → `fontSize: '11px'`; `grid-cols-[1fr_500px]` → `gridTemplateColumns: '1fr 500px'` (underscores are spaces).
- **Multi-declaration utilities**: many utilities emit several properties — `text-sm` sets `font-size` *and* `line-height`; `truncate` sets three; filter utilities (`brightness-[1.08] contrast-[.95]`) compose one `filter` value. Ground truth shows the full set; carry all of it.
- **Runtime-dynamic values** (`w-[${size}px]`, `style={{width}}`): a dynamic style function — arguments must be plain identifiers, the body a single object literal:

```ts
const styles = stylex.create({ bar: (width: number) => ({ width }) });
<div {...stylex.props(styles.bar(width))} />
```

- **Transforms**: Tailwind composes `scale-*`/`rotate-*`/`translate-*` through CSS variables into one `transform`; emit the single combined string, e.g. `transform: 'translateX(-0.5rem) rotate(3deg) scale(0.95)'`. Transitions become longhands (`transitionProperty`, `transitionDuration`, `transitionTimingFunction`); `transition-[width]` → `transitionProperty: 'width'`.
- **Gradients**: resolve `bg-gradient-to-r from-* to-*` to the emitted `background-image` value verbatim (v4 interpolates in oklab — the compiled value carries that).
- **Shadows/rings**: copy the emitted `box-shadow` verbatim; v4 renamed the shadow scale and changed ring defaults, so ground truth is the only safe source.

## Pseudo-elements

`before:`/`after:` become a **top-level `'::before'`/`'::after'` key inside the style namespace** (unlike pseudo-classes, which live inside property values). A generated pseudo-element needs explicit `content`:

```ts
const styles = stylex.create({
  item: {
    color: 'var(--foreground)',
    '::before': { content: '"*"', color: 'var(--live)' },
  },
});
```

## Animations

`animate-*` → `stylex.keyframes` + `animationName`, longhand animation properties. For Tailwind built-ins (`animate-spin` etc.), copy the `@keyframes` block out of `baseline.css` rather than recalling it:

```ts
const spin = stylex.keyframes({ from: { transform: 'rotate(0deg)' }, to: { transform: 'rotate(360deg)' } });
const styles = stylex.create({
  spinner: { animationName: spin, animationDuration: '1s', animationTimingFunction: 'linear', animationIterationCount: 'infinite' },
});
```

## Restructuring — utilities with no single-element home

StyleX styles one element ("all styles on an element are caused by class names on that element itself"). These utilities compile to descendant/sibling/ancestor selectors, so they change shape. Record every restructure in the migration report.

| Tailwind | Replacement |
| --- | --- |
| `space-x-*` / `space-y-*` | `gap` on the flex/grid parent; if the parent is neither, margins on the child styles |
| `divide-x-*` / `divide-y-*` | a border on each child after the first (`':first-child'` unsets it), or a separator element |
| `group` + `group-hover:` etc. | `stylex.when.ancestor(pseudo)` — mark the group element with `stylex.defaultMarker()` (or `defineMarker()` for independent nested groups), condition the child on the ancestor's state:<br>`<div {...stylex.props(styles.card, stylex.defaultMarker())}>` … child: `color: { default: 'gray', [stylex.when.ancestor(':hover')]: 'black' }` |
| `peer` + `peer-*:` | `stylex.when.siblingBefore(pseudo)` — mark the peer, condition the following sibling |
| `prose` (typography plugin) | `markdown.css` escape hatch — extracted from compiled output in P6; keep `not-prose` semantics by scoping the extracted selectors |
| `@container` / container-query variants | **unsupported in StyleX** (facebook/stylex#515) — escape-hatch CSS on bridge variables; flag it |
| `[&>svg]:` and other child-reaching arbitrary variants | style the child element directly, or relay through a CSS variable set conditionally on the parent |

Skip `stylex.when.descendant` / `anySibling` / `siblingAfter` for migration output — they compile to `:has()` and trip the `no-lookahead-selectors` lint rule. `ancestor` and `siblingBefore` are the safe pair.

The CSS-variable relay is the fully-stable alternative to `when.*` (works on any StyleX version): the parent sets a variable conditionally, the child consumes it —

```ts
// parent: { '--childColor': { default: 'gray', ':hover': 'black' } }
// child:  { color: 'var(--childColor)' }
```

## Expansion blocks

- `sr-only` expands to its full known declaration block (absolute position, 1px box, clip, nowrap, zero margin-workarounds) — copy from `baseline.css`.
- `container` is `width: 100%` plus per-breakpoint `maxWidth` conditions.
- `line-clamp-*` expands to the `-webkit-box` trio.

## Astro scoped `<style>` translation

`.astro` templates take the same resolve step, but the reshape target is a scoped `<style>` block: one semantic class per element role, declarations copied literally, colors/fonts/radii through the bridge variables, media queries verbatim from ground truth. Variants stay natural CSS (`.nav-link:hover { ... }`), which also makes `group`/`peer`-style patterns plain descendant CSS here — restructuring is only required on the StyleX side.
