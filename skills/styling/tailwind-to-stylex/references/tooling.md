# StyleX build wiring (P1)

Per-framework setup, plugin options, and the version check. The smoke test in SKILL.md P1 is the arbiter for everything here — wiring counts as working only when a real StyleX rule reaches the browser in dev **and** production build.

## Version check (run first)

This skill was written and tested against `@stylexjs/stylex` **0.19.0**. StyleX is pre-1.0 and its API moves (`stylex.attrs` was removed and restored; `when.*` is recent; plugin packages churn).

1. Pin exact versions on install (no `^`).
2. If the version you installed differs from 0.19.0, fetch the official LLM-oriented docs at https://stylexjs.com/docs/llm-resources (`stylex-installation.md`, `stylex-authoring.md`) and **prefer them over this skill's references wherever they conflict**.
3. Before relying on `stylex.when.*` or `defineConsts`, confirm they exist in the installed version.

## Two official integration paths

1. **`@stylexjs/unplugin`** — one package, targets: `stylex.vite()`, `stylex.webpack()`, `stylex.rspack()`, `stylex.esbuild()`, `stylex.rolldown()`. Preferred for plain Vite/Webpack/Rspack/esbuild apps.
2. **`@stylexjs/babel-plugin` + `@stylexjs/postcss-plugin`** — the babel plugin transforms `stylex.*` calls in source; the postcss plugin scans files and injects the generated CSS at a literal `@stylex;` at-rule in your global stylesheet. Preferred where a meta-framework owns the bundler (Next.js, Astro).

Either way there must be **one CSS entry file imported at the app root** for the generated CSS to land in.

Avoid `@stylexjs/nextjs-plugin` (stale, superseded by path 2). Community SWC compilers (`stylex-swc-plugin`, `@stylexswc/*`) exist and are faster, but are unofficial — fallback only, say so in the report.

## Babel plugin options that matter

```js
// babel config fragment
['@stylexjs/babel-plugin', {
  dev: process.env.NODE_ENV === 'development',
  runtimeInjection: false,              // must be false in production
  unstable_moduleResolution: { type: 'commonJS' }, // REQUIRED for defineVars/createTheme
  treeshakeCompensation: true,          // if the bundler tree-shakes token imports away
}]
```

`unstable_moduleResolution` may have been renamed post-0.19 — the version check covers this.

## PostCSS plugin

```js
// postcss.config.mjs
export default {
  plugins: {
    '@stylexjs/postcss-plugin': {
      include: ['src/**/*.{js,jsx,ts,tsx}'], // every file that calls stylex.*, including *.stylex.ts
      useCSSLayers: true,
    },
  },
};
```

And in the global stylesheet (while Tailwind coexists, keep both):

```css
@import "tailwindcss";
@stylex;
```

## Framework branches

### Astro (primary path: babel + postcss)

The only StyleX-in-Astro setup with evidence of working in dev and production builds. StyleX never styles `.astro` template syntax — only framework islands (`.tsx` etc.); `.astro` files use scoped `<style>` on the bridge variables (see SKILL.md P4).

1. Install `@stylexjs/stylex`, `@stylexjs/babel-plugin`, `@stylexjs/postcss-plugin`. Add the `postcss.config.mjs` above and `@stylex;` to the global CSS Astro already imports.
2. Wire the babel plugin into the island JSX pipeline. With `@astrojs/react`, check whether the installed version forwards a `babel` option to `@vitejs/plugin-react` (read its README/types); if yes:

```ts
// astro.config.ts
react({ babel: { plugins: [['@stylexjs/babel-plugin', { /* options above */ }]] } })
```

   If it forwards no babel option, add a Vite-level babel transform scoped to island files (`vite-plugin-babel`), or use the unplugin fallback below.
3. Run the smoke test with an island component on a real page.

**Fallback: unplugin's Vite target inside Astro.** Undocumented territory (no official Astro support exists). Model the wiring on the SvelteKit precedent — after the framework, `enforce` neutralized:

```ts
// astro.config.ts
import stylex from '@stylexjs/unplugin';
export default defineConfig({
  vite: { plugins: [{ ...stylex.vite({ useCSSLayers: true }), enforce: undefined }] },
});
```

Dev mode needs the virtual modules wired into the root layout head, gated to dev:

```astro
{import.meta.env.DEV && <link rel="stylesheet" href="/virtual:stylex.css" />}
{import.meta.env.DEV && <script type="module">{`import('virtual:stylex:runtime')`}</script>}
```

In production the plugin appends its CSS to an existing CSS asset (`cssInjectionTarget`) — a real global stylesheet import must exist or you get an orphaned `stylex.css` no page links. If neither official path smoke-tests clean, the community `@stylexswc/unplugin` and `unplugin-stylex` packages ship dedicated Astro entries (unofficial — flag in the report).

### Vite (plain Vite + React/Solid/etc.)

`@stylexjs/unplugin`'s `stylex.vite()` **before** the framework plugin (preserves Fast Refresh). Dev mode needs the same virtual-module `<link>`/`import()` pair in `index.html`, dev-gated. Vite dev is StyleX's roughest edge — smoke-test before trusting HMR.

### Next.js

Babel + postcss path per the official guide: babel plugin in `.babelrc.js`, postcss plugin with `include` globs covering `app/**` and/or `pages/**`, `@stylex;` in the root-imported global CSS. Works with Webpack and Turbopack on Next ≥ 16.0.3.

### Webpack / Rspack / esbuild

`@stylexjs/unplugin`'s matching target plus the loader/extract plugin the StyleX install docs list for that bundler (`MiniCssExtractPlugin`, `CssExtractRspackPlugin`; esbuild needs `metafile: true`).

## ESLint

Install `@stylexjs/eslint-plugin` and enable its recommended rules on `**/*.{ts,tsx,js,jsx}`. It mechanically catches most gotchas in `gotchas.md` (invalid shorthands, missing `default`, mixed `className` + `props`, lookahead selectors, tokens outside `.stylex.ts` files). Treat its errors as blocking from P4 onward.
