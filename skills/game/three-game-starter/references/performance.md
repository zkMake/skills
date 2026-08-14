# Performance

Ordered by payoff. The first three cost almost nothing to build and outweigh everything below them.

## 1. Cap the pixel count

The DPR budget in `core-runtime.md` is the single biggest win: a 3× DPR phone renders 9× the fragments of a 1× one for no visible improvement. Cap pixel ratio at ~1.5 *and* at a total-pixel budget (`sqrt(MAX_PIXELS / (w × h))`), then keep it adjustable at runtime so a slider can prove the cost of raising it.

Fragment-bound is the normal state of a stylized three.js game. Before optimizing anything else, halve the resolution and watch the frame time — if it collapses, you are fragment-bound and every fix belongs in this section (fewer full-screen passes, cheaper fragment shaders, smaller render targets), not in draw-call land.

## 2. Collapse the draw calls

Batch everything batchable (see `rendering.md`). One multi-draw call for hundreds of objects instead of hundreds of `drawElements`, plus none of the per-`Object3D` matrix-world, culling, sorting, and render-list cost. This is what makes frame time scale with *content* rather than with *object count*.

## 3. Download less

- KTX2 atlases instead of PNG/WebP: GPU-native, mipmapped, and grouped so a whole world fits inside the 16-texture-unit cap.
- Draco GLBs, with the uncompressed sources committed but never loaded.
- Per-scene asset opt-in — a manifest-derived required set, not the whole `assets/` folder.
- Code-split heavy modules so a scene fetches only the mechanics it contains.

All four are in `assets.md`. Measure the result: run a production build, read the actual byte sizes in `dist/assets`, and confirm in the Network tab that nothing unexpected is fetched.

## 4. Skip work that cannot be seen

A **sim window** is the cheapest large win in a level-shaped game: define a band around the player and let any ambient or mechanical system opt out of its tick entirely outside it.

```ts
const SIM_AHEAD = 45;
const SIM_BEHIND = 30;

export function overlapsSimWindow(rangeMin: number, rangeMax: number,
                                 playerZ: number,
                                 ahead = SIM_AHEAD, behind = SIM_BEHIND): boolean {
  return rangeMin <= playerZ + behind && rangeMax >= playerZ - ahead;
}
```

Both distances are per-caller overrides, because how far away a thing can freeze unnoticed depends on how big and how obviously *moving* it is: ambient detail can be cut close, a large machine whose parts stream continuously wants a wider berth.

One rule: a system that skips its tick must **not** integrate silently while frozen and then jump on re-entry. Either freeze the state or re-derive it from `time.elapsed` on the way back in. Put the shared window helper in one file and import it — every reimplementation drifts.

## 5. Stop allocating per frame

- Module-scope scratch `Vector3` / `Quaternion` / `Matrix4` reused across calls, never `new` inside `update()`.
- **Pool** anything spawned in bursts: debris chunks, impact decals, hit sprites, ripples. A pool is a fixed set of instances plus a free list; a burst reuses them. Allocating a `Points` object and a material per burst is the classic frame-hitch.
- One `setMatrix` per posed instance, not three property writes (see `rendering.md`).
- Never build a string in a hot path.

## 6. Trim the frame's fixed costs

- Shadows: one caster, shadow camera sized to the play area rather than the world, 1024² until a screenshot says otherwise.
- Postprocessing: each full-screen pass costs a full framebuffer read/write at the budgeted resolution. Prefer merged effects (the `postprocessing` package's `Effect`s) over separate passes, and render depth prepasses at reduced scale where the effect tolerates it.
- Put every discrete check on the 30 Hz `time.stepTick` rate rather than per frame — see the two rates in `core-runtime.md`.
- Keep debug-only dependencies behind dynamic `import()` so they are absent from the production bundle, and put them in their own chunk group in the bundler config — a UMD that self-invokes on evaluation must not sit in the first-load vendor bundle.

## Measure

Build a small perf monitor as a system, dynamically imported under `?debug` only:

- FPS and frame time, each as a ring buffer with a sparkline.
- CPU time — the span around `experience.tick()`.
- GPU time via `EXT_disjoint_timer_query_webgl2`, feature-detected and degrading to "unavailable" rather than throwing.
- `renderer.info`: draw calls, triangles, programs, geometries, textures.

The counters that catch real regressions are **draw calls** and **programs**. A draw-call count that climbs with content means something escaped the batch; a program count that climbs means a material is being recreated instead of reused.

Profile before and after every optimization, and report the numbers. An optimization without a measurement is a guess, and in three.js the guesses are usually wrong about which of the six sections above the bottleneck is in.
