# StyleX authoring rules and traps

Load before writing the first `stylex.create`; reload on any compiler or lint error. The ESLint plugin catches most of these mechanically — these are the rules behind its errors, plus traps it can't see.

## Authoring rules

- `stylex.create` at **module top level**, never inside render. Style values must be statically resolvable: literals, local constants, tokens imported from `.stylex.ts` files. Arbitrary function calls, spreads, computed keys, and values imported from ordinary modules all fail compilation.
- **camelCase properties** (`borderRadius`); `--custom-props` stay as-is.
- **Longhands only** for multi-value shorthands — the compiler/linter rejects them because they make merge order ambiguous:
  - `margin: '0 auto'` → `marginBlock: 0, marginInline: 'auto'`
  - `padding: '1rem 2rem'` → `paddingBlock: '1rem', paddingInline: '2rem'`
  - `border: '1px solid red'` → `borderWidth`, `borderStyle`, `borderColor`
  - single-value shorthands (`padding: 16`, `inset: 0`) are fine.
- **Numbers are px** for length properties (`width: 24` → `24px`); unitless properties (`lineHeight`, `opacity`, `zIndex`, `fontWeight`, `flexGrow`) take raw numbers; every other unit is a string (`'1.5rem'`, `'50%'`).
- **`default` is mandatory** on any property that has a condition key, at **every nesting level**; `default: null` when there is no base value. A missing default makes StyleX silently drop the condition — the single most common conversion bug.
- `null` as a value **unsets** the property (useful in variants).
- **Self-selectors only**: pseudo-classes, pseudo-elements, attribute selectors, and at-rules on the element itself. `'& .child'` and `'.dark &'` do not exist; cross-element styling goes through variable relays or `stylex.when.*` (see `mapping.md`).
- **Dynamic style functions**: plain-identifier arguments, body is a single object literal — no destructuring, defaults, statements, or `return`.

## Merge and application semantics

- `stylex.props(...)` merges **per property, last argument wins** — definition order and import order are irrelevant. `cn('base', active && 'on')` becomes `stylex.props(styles.base, active && styles.on)`; falsy arguments are skipped.
- Keep caller-supplied overrides **last** so consumers still win (this is the `twMerge` replacement).
- `stylex.props` output only styles **host elements** (`div`, `button`, …). Spreading it onto a custom component does nothing to that component's DOM — pass the style objects through a `style` prop and let the component apply them on its own host element:

```tsx
function Badge({ style }: { style?: stylex.StyleXStyles }) {
  return <span {...stylex.props(styles.base, style)} />;
}
```

- An element that spreads `stylex.props()` takes no separate `className`/`class` or `style` attribute (`no-conflicting-props`) — fold everything into the `props` call.
- Non-JSX-spread frameworks (Solid, Svelte, Vue, Qwik) use `stylex.attrs(...)` — same arguments, returns `class` + serialized `style` string to bind as attributes.

## Traps from real migrations

- **Barrel exports break the compiler.** Import `.stylex.ts` token files and shared style modules directly; a re-export through `index.ts` makes the static analyzer lose them and styles silently vanish.
- **Cross-file styles cost a little runtime.** Co-located `create`+`props` compiles away entirely; exported/imported style objects keep a small runtime merge. Fine for shared modules — just don't export styles that only one file uses.
- **Keyframes**: only `stylex.keyframes()` — a raw `@keyframes` string inside `create` fails. Keyframes are file-local; to share across files, wrap the name in a `defineVars` token.
- **Falsy values in style arrays** can upset TypeScript in some positions; prefer explicit ternaries (`cond ? styles.a : null`) when `&&` produces type errors.
- **Multi-declaration drops**: converting only the declaration you remember from a utility that emits several (`text-sm`, `truncate`, filters) — ground truth (`mapping.md`) is the defense.
- **Missing plugin ≠ broken code**: without the build integration wired, `stylex.props` returns nothing and every style "disappears" — that's a P1/tooling failure, not a conversion bug. Re-run the smoke test.

## Not supported — route to escape hatch, flag in report

- **Container queries** (facebook/stylex#515).
- **Descendant/complex selectors** — permanent design constraint, not a version gap.
- **Styling third-party DOM** you don't render (lazily-created dialogs, embeds) — escape-hatch CSS on bridge variables.

## Expectations to set in the report

- StyleX styling code runs roughly **2× the LOC** of the Tailwind it replaces for identical CSS output — expected, not a smell; the payback is typed tokens and compile-time failure of what Tailwind lets drift silently.
- StyleX is **pre-1.0**: versions are pinned exactly in P1; note the installed version in the report so future upgrades re-run the version check in `tooling.md`.
