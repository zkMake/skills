# Assets and the loading gate

Models, textures, fonts, audio registration, the **gate** that holds world construction until everything has landed, and per-scene opt-in so a scene downloads only what it uses.

## Resources

One class owns every loader and one `THREE.LoadingManager`. It extends `THREE.EventDispatcher` and emits `LOAD_START` / `LOAD_PROGRESS` / `LOAD_ERROR` / `ASSETS_LOADED`. Loaded assets land in a flat `items` record keyed `<basename><Suffix>` — `playerModel`, `noiseTexture`, `sunsetEnvMap`, `bodyFont` — so call sites read `resources.items.playerModel` with no path knowledge.

Discover files with `import.meta.glob`, never a hand-maintained list:

```ts
const modelsLocation = import.meta.glob("#assets/models/compressed/*.glb", { eager: true });
const texturesLocation = import.meta.glob("#assets/textures/compressed/*.ktx2", { eager: true });

const mapAssets = (entries, extension: RegExp, suffix: string) =>
  Object.entries(entries).map(([path, module]) => ({
    name: path.split("/").pop()!.replace(extension, "") + suffix,
    path: (module as { default: string }).default,
  }));
```

Split construction from start: the constructor computes the required set and registers audio with the manager; `beginLoading(renderer)` creates the loaders and starts them, because a KTX2 transcoder needs `detectSupport(renderer)` and the renderer does not exist yet in the constructor.

## The gate

`ASSETS_LOADED` is the single barrier the world builds on. `LoadingManager.onLoad` is **not** that barrier — three things can finish after it:

1. **KTX2 transcoding** happens in a worker, so `onLoad` can fire before the texture is in `items`. Track a `pendingCompressed: Set<string>` and clear each name in the load callback.
2. **Lazy code-split chunks** (see below) resolve on their own promise.
3. **Items registered late.** If audio `itemStart`s land in the constructor and every one of them completes during an `await` before the texture loaders register, the manager reaches `loaded === total` and fires `onLoad` with nothing loaded. Guard with a `loadingStarted` flag and ignore `onLoad` until the real loaders have started.

```ts
private maybeFinishLoading() {
  if (this.finished || !this.managerLoaded ||
      this.pendingCompressed.size > 0 || this.pendingChunks) return;
  this.finished = true;
  this.dispatchEvent({ type: EVENTS.ASSETS_LOADED });
}
```

Every one of those three conditions calls `maybeFinishLoading()` when it clears. A transcode **error** must delete its pending entry too, or a single bad file wedges the gate forever.

## Models — Draco

Author uncompressed GLBs into `assets/models/src/`, commit them, and never load them at runtime. A script compresses them into `assets/models/compressed/`, which is the only set the glob picks up. Commit both.

The pipeline, one `gltf-transform` op per call, chained through a temp dir:

```
dedup → prune → flatten → weld → reorder --target size → draco
```

Draco is **last** — it needs welded, indexed geometry — and applies its own 16-bit quantization, so there is no separate `quantize` pass (that would double-quantize via `KHR_mesh_quantization`). Print a `before → after` KB line per model so the win is visible.

At runtime: `DRACOLoader` with `setDecoderPath("draco/")`, wired into `GLTFLoader` via `setDRACOLoader`. Serve the decoder out of the installed `three` package with a small Vite plugin (dev middleware + `emitFile` at build) rather than committing a `public/` snapshot — `basis_transcoder` in particular is API-coupled to the loader version and drifts silently on upgrade.

Also add a `transform` plugin that rewrites three's module-level `new URL('../libs/draco/…', import.meta.url)` fallbacks to the runtime paths. Left alone, the bundler emits ~1.9 MB of decoder copies into `dist/assets` that are never fetched.

## Textures — KTX2 atlases

Ship compressed atlases, not source images. Pack related textures into a square `gridDim × gridDim` atlas; a cell can itself nest a 2×2 of four half-size textures. Sample by normalized UV offset (0.5 steps for a 2×2, 0.25 for nested sub-cells) so re-sizing cells needs no shader change.

Two config files, deliberately separate:

- **runtime** (`texture-atlas-config.ts`): `fileBase`, `itemKey`, `cellSize`, `encodeFormat` (srgb/linear), `encoder`, `hasAlpha`, gutter px, wrap mode.
- **bake-only** (`texture-atlas-bake-config.ts`): which source image goes in which cell. Kept out of the runtime bundle.

Bake with `sharp` (composite the grid) then the `ktx` CLI:

| Encoder | Flags | Use for |
| --- | --- | --- |
| ETC1S | `--encode basis-lz --qlevel 255` | detailed color art — small |
| UASTC | `--encode uastc-ldr-4x4 --uastc-quality 3 --zstd 18` | gradients, normal maps, noise — ETC1S bands them |

Hard rules: **group atlases by encoder** (one GPU texture cannot mix ETC1S and UASTC); sRGB art uses the `_SRGB` format, not `_UNORM` with a conversion; data textures (normals, noise) are `_UNORM` + `--assign-tf linear` and must never be tagged sRGB; `--zstd` is invalid for ETC1S. Always `--generate-mipmap`.

Keep the bake **on request** — no `package.json` script. It needs external CLIs, it is slow, and it is not part of the dev loop. Sources stay committed as the bake's real input; editing a `.ktx2` does nothing.

Pack noise channels rather than shipping three grayscale textures — one RGB texture holding perlin in R, voronoi in G, a third pattern in B costs one texture unit instead of three. Batched-mesh shaders are capped at 16 texture units, and atlasing is what keeps a whole world inside that.

**Debug escape hatch.** Gate a second path on `?debug=textures` that loads the uncompressed sources and composites canvas atlases at runtime, so a tester can hot-swap a texture in the browser and see it live. Keep the flag **runtime**-read, not build-time, so the link works against any deployed build; keep the source glob behind a dynamic `import()` so a normal visitor never fetches it. Once the sources are baked into a runtime atlas, null their `.image` and `.source.data` (they were never uploaded, so do not `dispose()`).

## Per-scene opt-in

Downloading every asset for every scene is the default and it is wrong. Derive the required set from the content:

```ts
export function computeAssetRequirements(): AssetRequirements {
  const models = new Set(ALWAYS_LOADED_MODELS);
  const sounds = new Set(ALWAYS_LOADED_SFX);
  for (const entity of currentConfig.entities) {
    for (const m of ASSET_MANIFEST[entity.type].models) models.add(m);
    for (const s of ASSET_MANIFEST[entity.type].sounds) sounds.add(s);
  }
  return { models, fonts, sounds, presentTypes };
}
```

`ASSET_MANIFEST` is a per-entity-type table of what that type needs; `ALWAYS_LOADED_*` is the baseline every scene uses. `Resources` filters its globs against the result. The manifest lives with the content schema, not in the renderer, so an editor or validator can read it too.

Warn in DEV when a config names an asset key the manifest does not know — silence there means a scene ships with a missing model and fails at build time instead of load time.

## Lazy chunks

Some modules are heavy and rare: a mini-game, a boss, a cutscene. Make each its own chunk fetched only when the scene contains it.

```ts
const loaded: Partial<LazyClasses> = {};

export async function loadRequiredModules(): Promise<void> {
  const present = new Set(currentConfig.entities.map((e) => e.type));
  const loads: Promise<unknown>[] = [];
  if (present.has("bossArena")) {
    loads.push(import("./boss-arena/boss-arena-factory.ts")
      .then((m) => { loaded.BossArenaFactory = m.BossArenaFactory; }));
  }
  await Promise.all(loads);
}

export function getLazyClass<K extends keyof LazyClasses>(name: K): LazyClasses[K] {
  const cls = loaded[name];
  if (!cls) throw new Error(`[lazy] "${name}" not loaded — the gate must resolve first.`);
  return cls;
}
```

Kick this off from the `Resources` constructor so chunk fetches overlap the asset download, and gate `ASSETS_LOADED` on it. World construction then stays synchronous while still getting the classes.

Two rules keep the split from silently collapsing:

- Every other reference to a split class is **type-only** (`import type`). One value import pulls the chunk back into the main bundle.
- Always-loaded code must never `instanceof` a split class. Export `isBossArena(entity)` style type guards keyed on `entity.type` + `instance !== undefined` and use those.
- A failed chunk fetch is fatal, not silent: log it and dispatch `LOAD_ERROR`. Building a world without a factory it needs crashes harder and later.

## Fonts and audio

TTF glyphs for in-scene 3D text load through `TTFLoader` — which hardcodes a CDN import of `opentype.js`. Alias that URL to the bundled dependency in `vite.config.ts` so the artifact has zero external requests.

Audio does not go through a three loader. Register the required cues with the same `LoadingManager` via `itemStart` / `itemEnd` so audio counts toward the progress bar (see `audio.md`), and keep streamed music **out** of the gate: a streamed track never reaches `canplaythrough` until playback buffers, so gating on it hangs the loading screen forever.

## Loading overlay

Listen to `LOAD_PROGRESS` for the bar and `LOAD_ERROR` to surface a failure instead of hanging on a half-full bar. Under `?debug`, add a panel listing every item with its state — that is what turns "stuck at 80%" into a filename.
