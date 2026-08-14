# Rendering

How to get hundreds of objects into a handful of draw calls, how to branch their look inside one shader program, and how to stack postprocessing without losing the ability to boot without it.

## Pick the batching strategy

| Situation | Use | Why |
| --- | --- | --- |
| A handful of objects (< ~50) | plain `THREE.Mesh` | Nothing to win; keep it simple. |
| Many copies of **one** geometry | `THREE.InstancedMesh` | One draw call, one geometry. |
| Many objects across **many different** geometries | `THREE.BatchedMesh` | One draw call across *heterogeneous* geometries via an indirect buffer. |
| Thousands of tiny sprites | `THREE.Points` + a pooled attribute buffer | Per-particle allocation is the cost to avoid. |

`BatchedMesh` is the choice that unlocks a real level. One mesh per object hits three separate walls: hundreds of `drawElements` calls with their state changes; per-`Object3D` matrix-world recomputation, frustum culling, sort ordering and render-list insertion every frame; and material/program switches you cannot batch away. `BatchedMesh` renders every registered geometry and instance in a single multi-draw call, culls per-instance internally, and skips the scene-graph cost.

## CachedBatchedMesh

Wrap `THREE.BatchedMesh` to add a string-keyed geometry registry, so a factory can register by name and dedupe:

```ts
class CachedBatchedMesh extends THREE.BatchedMesh {
  geometries = new Map<string, number>();      // key → geometryId
  geometryKeyById = new Map<number, string>();

  registerGeometry(geometry: THREE.BufferGeometry, key: string,
                   reservedVertexCount?: number, reservedIndexCount?: number): number {
    const existing = this.geometries.get(key);
    if (existing !== undefined) return existing;
    const id = this.addGeometry(geometry, reservedVertexCount, reservedIndexCount);
    this.geometries.set(key, id);
    this.geometryKeyById.set(id, key);
    return id;
  }
}
```

The optional `reserved*Count` args allocate slack beyond the actual geometry so a slot can be hot-swapped later with `setGeometryAt` without reallocating — that is how a glyph swaps to an outlined variant, or a prop to a damaged one.

Allocate one mesh per **material class**, not per object type. A typical split:

| Mesh | Ceiling (instances / geometry slots / vertices) | Holds |
| --- | --- | --- |
| `opaque` | 10000 / 1M / 1M | the bulk of the world |
| `transparent` | 2000 / 100K / 100K | fade-capable objects, `side: DoubleSide` |
| `decals` | 16 / 1K / 1K | pooled ripples/impacts, own layer, `frustumCulled = false`, high `renderOrder` |

Those ceilings are **budgets**. Overcommitting wastes GPU memory; undercommitting breaks world construction at runtime. Whenever an authoring limit is derived from one (max chain length, max spawn count), write the arithmetic in a comment next to both numbers — moving either one silently invalidates the other.

## Instances are not objects

An instance wrapper holds an `instanceId` and writes into shared buffers:

```ts
class BatchedInstance {
  constructor(mesh: CachedBatchedMesh, geometryId: number) {
    this.instanceId = mesh.addInstance(geometryId);
  }
  // position/quaternion/scale are observable proxies: a write marks dirty,
  // and a single setMatrixAt flushes it. No Object3D, no world-matrix graph.
}
```

Consequences to design around:

- **Pose with one `setMatrix` call.** Setting position, then quaternion, then scale through property setters costs three `setMatrixAt` round trips. Anything animating hundreds of instances per frame composes into a reusable module-scope `THREE.Matrix4` from module-scope scratch vectors and writes once.
- **Dispose by hand.** Instances are not `Object3D`s, so nothing in the scene graph frees them. Teardown must null the owning object's `instance` field explicitly to break the reference.
- **Per-instance color is a free channel.** `setColorAt` is often used to carry a type ID into the shader rather than an actual color.

## One shader, many looks

With everything in one material, variant logic lives inside the shader, keyed off per-instance data. Inject with `onBeforeCompile` and a chunk registry that factories contribute to:

```ts
const addShaderChunk = (chunks: ShaderChunks, slot: SlotName, chunk: string) => {
  (chunks[slot] ??= []).push(chunk);
};

material.onBeforeCompile = (shader) => {
  Object.assign(shader.uniforms, this.uniforms, { ...TYPE_IDS });
  shader.vertexShader = shader.vertexShader
    .replace("#include <common>", `#include <common>\n${chunks.vertexCommon.join("\n")}`)
    .replace("#include <begin_vertex>", `#include <begin_vertex>\n${chunks.vertexTransform.join("\n")}`);
  shader.fragmentShader = /* same pattern */;
};
```

Rules that keep this maintainable:

- Name the slots (`vertexCommon`, `vertexTransform`, `mapFragment`, …) and document each one's contract. An unnamed injection point becomes unfixable.
- Every factory contributes GLSL from its **own** `src/shaders/<name>/` folder, imported `?raw` (or via `vite-plugin-glsl` for `#include` support). No inline template-literal shaders.
- Give the material an `isDepthPass` variant if the object also renders into a depth prepass — the branch has to match or shadows and depth-based effects disagree with the color pass.
- When JS and GLSL both compute the same motion (a door swing driving both visuals and collision), factor the waveform into one function per language and comment each as the other's mirror. They drift the moment they are not called out.

## In-scene text

Batch glyphs through their own `CachedBatchedMesh` on a `MeshBasicMaterial`, one instance per letter, with per-instance alpha carried in a small `DataTexture` uniform. A word is then a set of instance transforms, not a mesh per glyph.

For an outline pass, reserve extra vertex/index slack at registration and swap the glyph geometry with `setGeometryAt`, rather than allocating a second mesh.

## Postprocessing

Use the `postprocessing` package's `EffectComposer` (effects merge into fewer passes than raw three.js passes). A workable stack, in order: custom depth/scene prepass targets → depth-driven effects (cheap DOF) → tone mapping → hue/saturation → SMAA → vignette → screen transition.

Two structural rules:

- **The composer is optional at boot.** Build it only when its dependencies exist (a water render target, a specific system), and keep the renderer's plain-render fallback for the frames before that. This is also what lets the whole pipeline be a system the user can switch off.
- **The composer is not in the scene graph.** Dispose it — and every render target — explicitly, *before* disposing scene materials, or shader programs are not reliably released.

Each effect is its own file extending `Effect`, with its own debug folder. Effects that own an animation (an intro reveal, an outro swirl) dispatch a window event on completion rather than reaching into game logic, so the sequencing stays in `main.ts`.

## Shadows and lighting

`VSMShadowMap` gives soft shadows cheaply on a stylized scene. Cast from exactly one directional light, size its shadow camera to the actual play area (not the world), and keep the map at 1024² until a screenshot proves it needs more. Prefer a baked or IBL ambient contribution over a second real-time light.

## Layers

Reserve a `THREE.Layers` bit per render pass that needs isolation — a water prepass, a decal pass, a thumbnail capture. A `LayerScopedRenderPass` that flips the camera's layer mask around one render is cheaper and far easier to reason about than toggling `visible` on dozens of objects.
