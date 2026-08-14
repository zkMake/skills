# Debug tooling

Tuning a game is thousands of small numeric decisions. The tooling that makes those cheap is worth building on day one, and it must cost production nothing.

## Flags

Read the URL once, at module scope, and export consts. Every gate reads the const — no repeated `URLSearchParams` parsing scattered through the code.

```ts
const params = new URLSearchParams(window.location.search);
const debugParam = params.get("debug");

export const DEBUG_ACTIVE = debugParam !== null;
export const DEBUG_TEXTURES_ACTIVE = debugParam === "textures";
export const CAPTURE_ACTIVE = params.get("capture") !== null;
```

Keep flags **runtime**-read rather than build-time where you can afford it: `?debug` then works against any deployed build, so a tester needs a link and nothing else. Pay for that with a dynamic `import()` on the heavy debug path so a normal visitor never fetches it.

## The pane

One `Debug` class, constructed first in `Experience` so any later constructor can add a folder:

```ts
class Debug {
  active = DEBUG_ACTIVE;
  ui!: Pane;

  constructor() {
    if (!this.active) return;
    this.ui = new Pane();
    // tweakpane styles its own wrapper (.tp-dfwv), not pane.element — style the
    // wrapper so a tall pane scrolls instead of running off-screen.
    const wrapper = this.ui.element.closest<HTMLElement>(".tp-dfwv") ?? this.ui.element;
    wrapper.style.maxHeight = "calc(100vh - 80px)";
    wrapper.style.overflowY = "auto";
  }
}
```

Each system adds its own folder in its own constructor, guarded, collapsed by default:

```ts
createDebugControls() {
  const { debug } = Experience.get();
  if (!debug.active || !debug.ui) return;
  const folder = debug.ui.addFolder({ title: "Water", expanded: false });
  folder.addBinding(this.params, "speed", { min: 0, max: 4 })
    .on(EVENTS.TWEAKPANE_CHANGE, (e) => { this.material.uniforms.uSpeed.value = e.value; });
}
```

Conventions worth enforcing: the folder lives with the system it controls, never in a central debug file; a control that should survive a reload goes through one shared `debug-preferences.ts` (localStorage read/write wrapped in try/catch for privacy mode) rather than inline keys; and `dispose()` removes the folder.

Tweakpane also earns a **"do X now" button** per timed or random behaviour — fire this emitter, strike now, spawn a burst. Waiting out a random schedule to test one code path is the biggest hidden tax in game work.

## Dev-tools panel

Once there is more to show than sliders, add an HTML panel with tabs, lazy-imported from `main.ts` under `DEBUG_ACTIVE` and disposed before any re-init:

| Tab | Contents |
| --- | --- |
| General | scene/entity picker, state readout, teleport, reset |
| Performance | the perf HUD (see `performance.md`) |
| Textures | every source texture with upload / preview / revert / download |
| Audio | bus sliders, soundscape picker, per-emitter fire buttons, live counts |
| Loading | every asset with its load state |

The **Textures** tab is the one that changes who can work on the game: a non-technical tester opens a deployed build with `?debug=textures`, uploads a candidate image, and sees it live in the scene. That requires the uncompressed-source + runtime-canvas-atlas path in `assets.md`, because a baked KTX2 cell cannot be swapped at runtime. Preview is uncompressed, so it shows art and fit, not the final compressed look — say so in the panel. The handoff is: tester downloads the approved file, it replaces the source, the atlas is re-baked.

The **Loading** tab is what turns "stuck at 80%" into a filename.

## Capture mode

A `?capture=<target>` flag that boots a stripped world, renders specific frames, and parks results on `window.__capture` for a headless driver (Playwright or an agent-browser CLI) to collect. Uses:

- Thumbnails of every entity/prop for an editor palette, committed as PNGs.
- Reference screenshots for visual regressions.

Two rules make it usable: the capture branch returns from `World`'s build **before** constructing decor and gameplay systems, so a thumbnail shows the subject and not the scenery; and every system that a capture must hide or pose exposes a small explicit hook (`setCaptureVisible(v)`) rather than the driver reaching into internals. Capture needs real GL, so it runs locally, not in CI.

## Bundle hygiene

Debug tooling must not reach production. Verify rather than assume:

- Debug-only deps (`OrbitControls`, GPU-memory shims, perf libs) are reachable only through dynamic `import()`.
- Give them their own chunk group in the bundler config so they never land in the first-load vendor bundle. A UMD that patches `getContext` on evaluation is exactly the thing that must stay in an async chunk.
- After a production build, grep `dist/assets` for the debug dep names and confirm they are only in async chunks. Then load the build with no flag and confirm the Network tab shows no debug fetch.
