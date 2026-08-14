# Core runtime

The non-optional foundation: bootstrap, the `Experience` singleton, the clock, the frame, the `World`, and teardown. Build all of it before any system.

## The frame

The renderer drives everything. One `setAnimationLoop` callback per frame, in this order, and nothing else starts a loop of its own:

```
renderer.setAnimationLoop(update)
└─ experience.tick()
   ├─ time.update()            advance the clock; set delta, elapsed, stepTick
   ├─ slowMotion?.update()     may rewrite time.timeScale — before anything reads delta
   ├─ camera.update()          follow / parallax, using this frame's delta
   └─ world.update()           every system, in the documented order
└─ postprocessing.render() ?? renderer.render(scene, camera)
```

A system participates by exposing `update()` and being called from `World.update()`. It never registers its own `requestAnimationFrame`, and never reads `performance.now()` — one clock, or a time scale tears the world apart.

The frame runs at **two rates**, and choosing wrongly is the most common perf mistake in this architecture:

| Rate | Signal | For |
| --- | --- | --- |
| Every frame | `time.delta` | anything the eye tracks: motion, animation, shader uniforms, camera |
| Fixed ~30 Hz | `if (!time.stepTick) return;` | discrete checks: AABB refresh, collision, overlap and trigger tests, support queries |

Anything whose answer does not change visibly between two frames belongs on `stepTick`. A third rate is optional and cheap: a **sim window** lets a system skip its whole tick when it is too far from the player to be noticed (`performance.md`).

## File tree

```
index.html
package.json
vite.config.ts
tsconfig.json
src/
  main.ts                        bootstrap: canvas, config ingestion, fatal errors
  style.css
  config/config-manager.ts       the active scene/level config (skip if hand-placed)
  experience/
    experience.ts                the singleton root
    eventTypes.ts                EVENTS enum — every event name in the app
    core/
      sizes.ts                   viewport + DPR budget, dispatches RESIZE
      time.ts                    delta / elapsed / timeScale / step accumulators
      camera.ts                  camera + follow rig (+ orbit under ?debug)
      renderer.ts                WebGLRenderer, owns the animation loop
      resources.ts               loaders + the loading gate
      debug.ts                   tweakpane pane, gated on ?debug
      debug-flag.ts              DEBUG_ACTIVE, read once from the URL
    utils/dispose-object3d-gpu.ts
    world/world.ts               builds, ticks, disposes every system
  assets/{models,textures,fonts}/
  shaders/<name>/{vertex,fragment}.glsl
```

Set the `#*` → `./src/*` import alias in `package.json` `imports` (or `tsconfig` paths) and use it everywhere — relative `../../..` chains rot the moment a file moves.

## Experience

One singleton, constructed once, owning every core object. Systems reach each other through `Experience.get()` instead of threading references through constructors.

```ts
export class Experience {
  private static _instance: Experience | null = null;

  canvas: HTMLCanvasElement;
  debug: Debug;
  /** Abort to detach every DOM listener registered for this instance. */
  domListenerAbortController = new AbortController();
  readonly domListenerSignal = this.domListenerAbortController.signal;
  sizes: Sizes;
  time: Time;
  slowMotion?: SlowMotion; // only if time control is on the build sheet
  scene: THREE.Scene;
  resources: Resources;
  camera: Camera;
  renderer: Renderer;
  world: World;

  static init(canvas: HTMLCanvasElement) {
    this._instance?.destroy(); // re-init replaces, never stacks
    return new Experience(canvas);
  }

  static get() {
    if (!this._instance) throw new Error("Experience not initialized.");
    return this._instance;
  }

  private constructor(canvas: HTMLCanvasElement) {
    Experience._instance = this;
    this.canvas = canvas;
    this.debug = new Debug();
    this.sizes = new Sizes(this.domListenerSignal);
    this.time = new Time();
    this.scene = new THREE.Scene();
    this.resources = new Resources();
    this.camera = new Camera();
    this.renderer = new Renderer();
    this.world = new World();
    this.resources.beginLoading(this.renderer.instance); // needs the GL context
    this.sizes.addEventListener(EVENTS.RESIZE, this.onResize);
  }

  private onResize = () => {
    this.camera.resize();
    this.renderer.resize();
  };

  /** One frame, before rendering. Order per "The frame" above. */
  tick() {
    this.time.update();
    this.slowMotion?.update();
    this.camera.update();
    this.world.update();
  }

  destroy() { /* see Teardown below */ }
}
```

Construction order is load-bearing: `Debug` before anything that registers a debug folder, `Sizes` before `Renderer`, `Renderer` before `resources.beginLoading` (a KTX2 transcoder needs `detectSupport(renderer)`), `World` last so its constructor can subscribe to `Resources`.

## Time

One clock, scaled, producing the two rates above.

```ts
const STEP_S = 1 / 30; // collision / AABB refresh rate

class Time {
  start = performance.now() * 0.001;
  current = this.start;
  elapsed = 0;
  delta = 0.016;
  /** 1 = real time. A slow-motion controller drives this. */
  timeScale = 1;
  /** True on frames where collision bounds should be refreshed (~30 Hz). */
  stepTick = false;
  private accumulator = STEP_S;

  update() {
    const now = performance.now() * 0.001;
    this.delta = (now - this.current) * this.timeScale;
    this.current = now;
    // elapsed accumulates the SCALED delta, not wall-clock-minus-start, so
    // shader uTime and every phase clock slow in lockstep with delta.
    this.elapsed += this.delta;

    this.accumulator += this.delta;
    this.stepTick = false;
    if (this.accumulator >= STEP_S) {
      this.accumulator -= STEP_S;
      this.stepTick = true;
    }
  }
}
```

Clamp `delta` (e.g. `Math.min(delta, 0.1)`) if a tab-away frame would let anything integrate through a wall.

## Sizes and the DPR budget

`Sizes` owns width, height, and pixel ratio, and re-dispatches `resize`. The pixel ratio is a **budget**, not `devicePixelRatio`:

```ts
const MAX_PIXEL_RATIO = 1.5;
const MAX_PIXELS = 1_650_000; // ~1600×1030 of real fragments

const computePixelRatio = (w: number, h: number, budget: boolean) => {
  const caps = [window.devicePixelRatio, MAX_PIXEL_RATIO];
  if (budget) caps.push(Math.sqrt(MAX_PIXELS / (w * h)));
  return Math.max(1, Math.min(...caps));
};
```

This is the single highest-leverage performance knob in a three.js game: a 3× DPR phone renders 9× the fragments of a 1× one for no visible gain. Register the resize listener with `Experience.domListenerSignal` so teardown detaches it.

## Renderer

Where the frame above is implemented.

```ts
createInstance() {
  this.instance = new THREE.WebGLRenderer({ canvas: this.canvas, antialias: false });
  this.instance.setSize(this.sizes.width, this.sizes.height);
  this.instance.setPixelRatio(this.sizes.pixelRatio);
  this.instance.setAnimationLoop(this.update);
}

update = () => {
  this.monitor?.begin();
  this.experience.tick();
  if (this.postprocessing) this.postprocessing.render();
  else this.instance.render(this.scene, this.camera.instance);
  this.monitor?.end();
};
```

`antialias: false` because postprocessing (SMAA) or the DPR budget handles it more cheaply. Keep the plain-render fallback: the composer may not exist for the first frames.

In `dispose`, call `setAnimationLoop(null)` then `instance.dispose()`. **Never** `forceContextLoss()` — a re-init reuses the same canvas and would build a renderer on a dead context.

## Events

One `EVENTS` enum, every name in it, no string literals at call sites. It doubles as the app's event index.

```ts
const EVENTS = {
  // DOM
  KEYDOWN: "keydown",
  POINTER_MOVE: "pointermove",
  RESIZE: "resize",
  // tweakpane
  TWEAKPANE_CHANGE: "change",
  // resources.ts
  ASSETS_LOADED: "assetsLoaded",
  LOAD_START: "loadStart",
  LOAD_PROGRESS: "loadProgress",
  LOAD_ERROR: "loadError",
  // add one commented group per system
} as const;
```

Systems that emit extend `THREE.EventDispatcher<EventMap>` with a typed map, so listeners get payload types.

## World

`World` is the only place that knows which systems exist. It builds them on the gate, ticks them in a deliberate order, and disposes them.

```ts
class World {
  lighting!: Lighting;
  player?: Player;
  water?: Water;

  constructor() {
    const experience = Experience.get();
    this.scene = experience.scene;
    this.resources = experience.resources;
    this.resources.addEventListener(EVENTS.ASSETS_LOADED, this.onResourcesReady);
  }

  onResourcesReady = () => {
    this.lighting = new Lighting();
    this.player = new Player();
    if (WATER_ENABLED) this.water = new Water();
    this.configureListeners();
  };

  update() {
    this.lighting?.update();
    this.player?.update();
    this.water?.update();
  }

  dispose() {
    this.resources.removeEventListener(EVENTS.ASSETS_LOADED, this.onResourcesReady);
    this.water?.dispose();
    this.player?.dispose();
  }
}
```

Two rules that pay off later:

- **Tick order is documented, not incidental.** When one system reads state another writes this frame (a hit reaction reading the player's position), comment the reason at the call site — otherwise someone reorders it and gets a one-frame lag bug.
- **Keep `onResourcesReady` synchronous.** Anything that must be loaded first belongs behind the gate in `Resources`, not in an `await` here; downstream code relies on the world existing the moment the event returns.

## Bootstrap (`main.ts`)

```ts
const canvas = document.querySelector<HTMLCanvasElement>("canvas.webgl");
if (!canvas) throw new Error("Canvas element `.webgl` not found");

function run(config: SceneConfig | null) {
  if (config) ConfigManager.initFromConfig(config);
  Experience.init(canvas);
  if (DEBUG_ACTIVE) void import("#experience/world/overlay/dev-tools-entry.ts")
    .then(({ mountDevTools }) => { devTools = mountDevTools(); });
}
```

If a scene config can arrive from more than one source — baked in, a URL parameter, a fetched file — branch here and keep each branch behind a **dynamic import**, so the unused branch's dependency graph is eliminated from the bundle. Refuse to build a world from a config you could not validate: render a fatal-error overlay naming the reason instead of crashing mid-build.

## Camera

Wrap the camera in a class exposing `instance`, `resize()`, `update()`, `dispose()`. A follow camera lerps toward a target offset from the player each tick. Two additions worth having from day one:

- **Parallax** — parent the camera to a group that translates a few tenths of a unit on pointer-move. Cheap depth. Skip its update while orbit controls are active: it corrupts their world-space math.
- **Orbit under `?debug`** — lazy-import `OrbitControls` only when debug is on, so `three/examples` stays out of the production bundle.

## Teardown

A re-init that leaks is the default outcome unless teardown is written first. Order matters — composer and render targets before scene materials, renderer last.

```ts
destroy() {
  this.domListenerAbortController.abort();     // every DOM listener at once
  this.sizes.removeEventListener(EVENTS.RESIZE, this.onResize);
  this.world.dispose();                        // systems: tweens, listeners, GPU
  this.camera.dispose();
  this.renderer.disposePostprocessingPipeline(); // not in the scene graph
  const disposed = new Set<THREE.Texture>();
  disposeObject3DGpuResources(this.scene, disposed);
  disposeSceneBackgroundAndEnvironment(this.scene, disposed);
  this.renderer.disposeWebGLRenderer();
  this.resources.dispose(disposed);
  this.debug.ui?.dispose();
  Experience._instance = null;
}
```

`disposeObject3DGpuResources(root, disposedTextures)` walks the tree disposing geometries, materials, and every texture-valued material property, taking a shared `Set` so a texture reused by ten materials disposes once. Pass the same set into `Resources.dispose` so loaded items are not double-disposed.

The standing rules for every system:

- Allocate a GPU resource → dispose it in your own `dispose()`.
- Animate with GSAP → `gsap.killTweensOf(target)` in `dispose()`. A global timeline outlives the `Experience`; never leave it scaled or running.
- Register a DOM listener → pass `{ signal: Experience.get().domListenerSignal }`.
- Add a `setTimeout` / `setInterval` → clear it in `dispose()`.

## Verify

`tsc --noEmit` plus lint on every change. Boot the dev server, confirm a rendered frame and a clean console, then re-init three times and confirm GPU memory returns to its baseline.
