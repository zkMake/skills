---
name: tailwind-to-stylex
description: Migrate a Tailwind CSS v4 codebase to StyleX through a phased, visually verified workflow that ends pixel-identical and token-driven.
disable-model-invocation: true
---

# Tailwind v4 → StyleX migration

Migrate a Tailwind CSS v4 codebase to StyleX so the site renders pixel-identical while gaining typed, compiler-checked styles. Work the seven phases in order; Tailwind and StyleX coexist until the final teardown. Verification — screenshots plus computed styles — decides whether work passed, never judgment.

## Non-negotiables (every phase)

- **Ground truth**: resolve every utility from the project's own compiled Tailwind CSS captured in P0, never from memorized tables. Models hallucinate v3 values under v4 (ring width/color, shadow scale renames, oklch palette, `color-mix()` opacity modifiers). A class missing from the ground-truth CSS gets rendered once and read via computed style, then flagged in the report.
- **Literal-then-lift**: P4 translates values literally so fidelity is checkable; P5 lifts to idiomatic StyleX afterward. Lift only in P5.
- **Escape hatch**: exactly three sanctioned plain-CSS homes — `reset.css` (P3), the token bridge variables (P2), and `markdown.css` for subtrees StyleX cannot style (rendered markdown/`prose`, third-party DOM, container queries). Each consumes the `defineVars` CSS variables. In Astro, scoped `<style>` blocks are a fourth first-class home for `.astro` templates. Plain CSS anywhere else is a defect to fix.
- **Diff-clean gate**: a unit of work is done only when verification reports no visual or computed-style difference, or every difference is explained in the report and accepted. Never commit past an unexplained diff.

## P0 — Baseline

Load `references/verification.md` now.

1. Create a migration branch. Create a gitignored `migration/` working directory.
2. Inventory the codebase into `migration/manifest.md`:
   - Tailwind version and wiring. This skill targets v4; on v3, stop and report that the project should upgrade to v4 first (or accept degraded accuracy).
   - Framework(s): plain JSX app, Next.js, Astro (count `.astro` templates vs framework islands), etc.
   - `@theme` / `@theme inline`, `@custom-variant`, `@utility`, `@plugin` contents; `:root`/`.dark`-style variable blocks; dark-mode strategy (class, data-attribute, or media).
   - Dynamic class construction: `cn()`/`clsx`/`twMerge`/`cva`, `class:list`, template literals, shared class-string modules.
   - Restructuring census — grep for: `prose`, `space-x-`/`space-y-`, `divide-`, `group`/`group-`, `peer`, `@container`/`@sm`/`@lg`, `animate-`, `@apply`. Also note dead utilities (e.g. `group` with zero `group-*:` consumers) — those get deleted, not migrated.
   - Every file containing Tailwind classes, as a checklist ordered leaf components first, layouts last.
3. Capture ground truth: build the project once and copy the emitted CSS asset(s) to `migration/baseline.css` (fallback: `npx @tailwindcss/cli -i <global.css> -o migration/baseline.css`).
4. Capture the visual baseline per `references/verification.md`: full screenshot matrix plus computed-style dumps.

**Done when** the manifest lists every Tailwind-bearing file and `migration/` holds `baseline.css`, the screenshot set, and the style dumps.

## P1 — Tooling

Load `references/tooling.md` now; it carries per-framework wiring (Astro included), plugin options, and the version check against live StyleX docs (StyleX is pre-1.0; APIs move).

1. Install `@stylexjs/stylex` (exact pin) plus the integration `tooling.md` prescribes for this framework, and `@stylexjs/eslint-plugin`.
2. Run the version check from `tooling.md`.
3. **Smoke test**: add a throwaway component styled with one distinctive `stylex.create` rule, render it on a real page, and confirm the rule reaches the browser in `dev` mode **and** in the production build output. Then remove it.

**Done when** the smoke test passes in both modes. If it cannot be made to pass, stop and report — never start migrating on broken tooling.

## P2 — Tokens

Load `references/tokens.md` now.

1. Port every `@theme` token and hand-written theme variable into `defineVars` (themeable values: colors, fonts, radii) and `defineConsts` (static values: breakpoints, z-index, durations) in `.stylex.ts` files.
2. Use literal `--`-prefixed keys so escape-hatch CSS and StyleX share one variable namespace (the bridge).
3. Encode dark mode per `tokens.md` (`light-dark()` when the project controls `color-scheme`; `createTheme` otherwise).
4. Delete the old variable definitions the tokens replace in the same commit, re-pointing whatever consumed them.

**Done when** every theme token has exactly one home, the build passes, and the full route matrix is diff-clean.

## P3 — Reset

1. Extract Tailwind Preflight's rules verbatim from `migration/baseline.css` into `src/styles/reset.css` and import it at the app root. Verbatim extraction — not a lookalike reset — is what keeps the baseline identical after teardown.
2. While Tailwind is still installed the rules exist twice; that is expected and harmless because they are identical.

**Done when** `reset.css` matches the Preflight block in `baseline.css` and the matrix is diff-clean.

## P4 — Migrate (component by component)

Load `references/mapping.md` and `references/gotchas.md` before the first component; return to `mapping.md` whenever a class carries a variant or resists direct mapping.

For each manifest entry, leaf-first:

1. Collect the **complete** class set per element, including every branch of `cn()` calls, ternaries, `class:list` arrays, and template literals — a conversion from a partial class set silently drops styles.
2. Resolve each class against `migration/baseline.css`; reshape per `mapping.md`:
   - JSX components (React/Preact/Solid/islands): `stylex.create` at module top, applied with `stylex.props` (JSX-spread frameworks) or `stylex.attrs` (Solid/Svelte/Vue/Qwik).
   - `.astro` templates: a scoped `<style>` block with semantic class names, declarations copied literally from ground truth, colors/fonts through the bridge variables.
   - Shared class-string modules (`cn`-based builders): one shared `stylex.create` module; parametrized builders become variant keys selected at the call site.
   - Caller `className` overrides become a style prop passed **last** to `stylex.props` (last wins, matching `twMerge` semantics).
3. Verify the routes that render the component; commit per component (Conventional Commits).
4. Record anything that needed restructuring, went to an escape hatch, or would not convert.

**Done when** every manifest entry is checked off, a grep of each migrated file finds zero Tailwind classes, and every route is diff-clean.

## P5 — Lift (global pass)

One pass over the whole migrated codebase, now that repetition is visible across it:

1. Replace repeated literal values with token references (`defineVars`/`defineConsts`); a value used once stays inline.
2. Rename mechanical style keys to semantic ones; factor styles shared across components into shared modules (import `.stylex.ts` files directly — barrel re-exports break the compiler).
3. Delete unused style keys and dead escape-hatch rules.

**Done when** the StyleX ESLint plugin reports clean, no literal value that has a token remains inline in more than one place, and the full matrix is diff-clean.

## P6 — Teardown

1. Extract the compiled `prose`/typography rules from `migration/baseline.css` into `markdown.css` **before** removing the plugin that generates them.
2. Remove Tailwind entirely: the `tailwindcss` package and its Vite/PostCSS wiring, plugins, `@import "tailwindcss"` / `@theme` / `@custom-variant` / `@plugin` lines, `tailwind-merge` and now-unused `cn()`/`clsx` helpers, and class-sorting formatter config (`prettier-plugin-tailwindcss` and editor/formatter class-function lists).
3. Prove removal: grep for `tailwind`, `@apply`, and a sample of utility classes across the repo — zero hits outside `migration/`.
4. Run the **full** verification matrix one final time against the P0 baseline, plus the project's build/typecheck/lint.
5. Write the final report: files converted, restructured patterns, escape-hatch contents and why each line is there, flagged non-conversions, and the LOC delta (roughly 2× styling LOC is normal for StyleX — expected, not a defect).

**Done when** the grep is clean, the final matrix is diff-clean against the P0 baseline, the build passes, and the report is delivered.

## References

| File | Load when |
| --- | --- |
| `references/verification.md` | P0, and at every diff-clean gate |
| `references/tooling.md` | P1 — build wiring per framework, version check |
| `references/tokens.md` | P2 — `@theme` → vars/consts, dark mode |
| `references/mapping.md` | P4 — variants, restructuring, everything past plain utilities |
| `references/gotchas.md` | Before the first `stylex.create`; again on any compiler or lint error |
