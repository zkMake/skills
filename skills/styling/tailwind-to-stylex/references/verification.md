# Verification: the diff-clean gate

What must be captured, when, and what counts as a pass. Requirements first — use whatever browser tooling is available (a browser-automation skill, MCP browser, or the Playwright fallback below); the requirements never change with the tool.

## The matrix

Capture, for **every route** the site serves (enumerate from the router/pages directory in P0):

- **Viewports**: 375×812 (mobile), 768×1024 (tablet), 1440×900 (desktop) — plus one extra width just below any custom breakpoint the project defines, if it isn't already straddled by these three.
- **Themes**: light and dark (skip dark only if the project has no dark mode).
- **Full-page** screenshots (not viewport-clipped), animations disabled (`prefers-reduced-motion` emulation or a global `*{animation:none;transition:none}` injection at capture time — the same injection for baseline and after, so it cancels out).

Store as `migration/shots/<phase>/<route>-<viewport>-<theme>.png` with a stable naming scheme so before/after pairs line up mechanically.

## Computed-style dumps

Screenshots can't capture hover/focus/stateful styling. For each route, dump `getComputedStyle` as JSON for:

- every interactive element (`a, button, input, select, textarea, [role], [tabindex]`), in default **and** hover **and** focus-visible state (drive the state, then dump);
- one representative element per migrated component (use a selector list built from the P0 manifest).

Store as `migration/styles/<phase>/<route>-<theme>.json`, keys ordered stably so plain `diff` works.

## When to capture

| Moment | Scope |
| --- | --- |
| P0 baseline | full matrix + full dumps (this is the reference forever — do it before touching anything) |
| After each P4 component | the routes that render it, both themes, all viewports |
| After P2, P3, P5 | full matrix |
| After P6 teardown | full matrix + full dumps, compared against **P0**, not against P5 |

## Pass criteria

- **Pixel diff**: compare each pair with a perceptual differ; pass at < 0.1% differing pixels with anti-aliasing tolerance on. Zero is the target; the tolerance absorbs font rasterization noise only.
- **Style diff**: the JSON dumps diff empty, except entries you can name and justify.
- **Explain-or-fix**: any diff above threshold is either fixed before committing, or written into the migration report with the exact cause and the user-visible consequence. "Probably fine" is not an explanation.

## Playwright fallback

Guaranteed-available path when no better browser tooling exists. One-time setup: `npm i -D playwright pixelmatch pngjs && npx playwright install chromium`.

```js
// migration/capture.mjs — node migration/capture.mjs <phase> <baseURL>
import { chromium } from 'playwright';
import fs from 'node:fs';

const [phase, base] = process.argv.slice(2);
const routes = JSON.parse(fs.readFileSync('migration/routes.json', 'utf8')); // ["/", "/about/", ...]
const viewports = { mobile: [375, 812], tablet: [768, 1024], desktop: [1440, 900] };
const themes = ['light', 'dark'];
const INTERACTIVE = 'a, button, input, select, textarea, [role], [tabindex]';

const browser = await chromium.launch();
for (const route of routes) {
  for (const [vp, [w, h]] of Object.entries(viewports)) {
    for (const theme of themes) {
      const page = await browser.newPage({ viewport: { width: w, height: h } });
      // class-strategy dark mode; for media strategy use page.emulateMedia({ colorScheme: theme })
      await page.addInitScript(t => localStorage.setItem('theme', t), theme);
      await page.goto(base + route, { waitUntil: 'networkidle' });
      await page.addStyleTag({ content: '*{animation:none!important;transition:none!important;caret-color:transparent!important}' });
      const slug = route.replaceAll('/', '_') || '_';
      await page.screenshot({ path: `migration/shots/${phase}/${slug}-${vp}-${theme}.png`, fullPage: true });
      if (vp === 'desktop') {
        const dump = {};
        for (const el of await page.locator(INTERACTIVE).all()) {
          const key = await el.evaluate(e => e.tagName + '#' + (e.id || e.className));
          dump[key] = { default: await el.evaluate(e => ({ ...getComputedStyle(e) })) };
          await el.hover().catch(() => {});
          dump[key].hover = await el.evaluate(e => ({ ...getComputedStyle(e) }));
        }
        fs.mkdirSync(`migration/styles/${phase}`, { recursive: true });
        fs.writeFileSync(`migration/styles/${phase}/${slug}-${theme}.json`, JSON.stringify(dump, Object.keys(dump).sort(), 1));
      }
      await page.close();
    }
  }
}
await browser.close();
```

Adapt the dark-mode init to the project's actual toggle (P0 inventory says which). Diff pairs:

```js
// migration/diff.mjs — node migration/diff.mjs <phaseA> <phaseB>
import pixelmatch from 'pixelmatch';
import { PNG } from 'pngjs';
import fs from 'node:fs';

const [a, b] = process.argv.slice(2);
let failures = 0;
for (const f of fs.readdirSync(`migration/shots/${a}`)) {
  const A = PNG.sync.read(fs.readFileSync(`migration/shots/${a}/${f}`));
  const B = PNG.sync.read(fs.readFileSync(`migration/shots/${b}/${f}`));
  if (A.width !== B.width || A.height !== B.height) { console.log(`SIZE ${f}`); failures++; continue; }
  const n = pixelmatch(A.data, B.data, null, A.width, A.height, { threshold: 0.1 });
  const pct = (n / (A.width * A.height)) * 100;
  if (pct >= 0.1) { console.log(`DIFF ${f}: ${pct.toFixed(3)}%`); failures++; }
}
console.log(failures ? `${failures} failures` : 'diff-clean');
```

Run the dev server (or `preview` after a build — prefer the built output, it's what ships) as the capture target, same mode for both sides of every comparison.
