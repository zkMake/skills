---
name: three-game-starter
description: Interview the user about the systems their game needs, then scaffold a pluggable three.js starter app on a shipped-game architecture.
disable-model-invocation: true
---

# Three.js game starter

Scaffold a new three.js game. Interview the user about which systems the game will carry, then build a starter where every system is pluggable and the core is the same core a shipped game runs on.

The architecture is not invented here — it is lifted from a production three.js game (batched single-draw-call rendering, KTX2 atlases, Draco models, per-scene asset opt-in, code-split heavy modules, bus-mixed audio, full teardown between runs). The reference files carry it. Your job is to pick the subset the user actually needs and wire it correctly.

## Vocabulary

These words name the architecture. Use them in the scaffold's code, comments, and your questions — they are what makes the pieces recognizable to the next agent.

- **Experience** — the singleton root. Owns every core object, reached everywhere via `Experience.get()`.
- **World** — owns the scene contents. Builds every system once assets land, ticks them, disposes them.
- **System** — one pluggable module: a class with a constructor, `update()`, `dispose()`, and an exported `ENABLED` flag. Adding or deleting one touches only `World`.
- **Tick** — one frame: `time.update()` → camera → world → render. Every system ticks from here; nothing runs its own loop.
- **Step** — the fixed ~30 Hz rate inside the tick (`time.stepTick`) that discrete checks run on instead of every frame.
- **Teardown** — `Experience.destroy()`: every GPU resource, listener, tween, and sound released so a re-init leaks nothing.
- **Gate** — the loading barrier. Nothing builds until every required asset AND every lazy chunk has landed.
- **Bus** — an audio mixer channel (`master` / `music` / `ambience` / `sfx`).
- **Batched** — many objects, one draw call.
- **Budget** — a preallocated ceiling (instance slots, pixel count, DPR) that authoring limits are derived from.
- **Sim window** — the Z/radius band around the player outside which a system skips its tick entirely.

## 1. Interview

Ask one round per message, in order. Put the default in parentheses on every question; accept "defaults" to take the rest of a round as written. Record every answer — step 2 reads them back.

**Round 1 — Shape and lifecycle**

- Project name and target directory?
- The core loop in one sentence?
- Camera: follow / fixed isometric / orbit / first-person (follow)?
- Stack: vanilla TS + three, or React Three Fiber (vanilla TS + three)?
- Toolchain: vite plus which package manager (vite + npm)?
- Does a page load run one session, or does the game re-init per level/run (re-init)? Re-init is what forces the teardown discipline.

**Round 2 — Content**

- What populates the world: a hand-placed scene, or data-driven from a level/config schema with validation (schema)?
- If data-driven, where does a config come from: baked in, a URL parameter, or a fetched file (baked in)?
- Peak simultaneous objects, and is it one repeated geometry or many different ones?
- Is content all built up front, or does it spawn and stream during play?
- Does the 3D scene render text (words, letters, labels)?

**Round 3 — Audio**

- One-shot SFX?
- Background music: none / one track / a shuffled rotation?
- Soundscape ambience — a looping bed plus randomized one-shot emitters?
- Which buses beyond `master`?
- Load only the cues the current scene needs, or all of them (only what's needed)?

**Round 4 — Physics and time**

- Physics: none / kinematic (tweens + gravity + AABB checks) / a full engine like Rapier (kinematic)?
- What blocks, carries, or kills the player?
- Does the game need slow motion or any global time scale?

**Round 5 — Visuals and assets**

- Postprocessing? Which effects (tone mapping, DOF, vignette, SMAA, custom)?
- Custom GLSL shaders? Shadows? Env map / IBL? Water, sky, or weather?
- Loading overlay, plus intro/outro screen transitions?
- Models: GLB through Draco (yes)? Textures: KTX2 compressed atlases or plain images (atlases)? Fonts in-scene?
- Any module heavy enough to code-split and fetch only when the scene needs it?

**Round 6 — Tooling and performance**

- Tweakpane debug panel behind a `?debug` URL flag (yes)?
- Perf HUD (FPS / CPU / GPU / draw counts)?
- Headless screenshot capture for asset thumbnails or regression shots?
- Target device class and frame budget — what has to hold 60fps?

Done when: every round has an answer, and no answer is a guess you made on the user's behalf.

## 2. Build sheet

Write the answers back as a build sheet: the stack, the systems being built, the systems deliberately left out, and the file tree. Name the reference file each system comes from. Ask the user to confirm or amend it.

Done when: the user has confirmed the build sheet.

## 3. Scaffold the core

Load `references/core-runtime.md` and build everything in it. The core is not optional — every branch of the interview sits on it.

Done when: the app boots to a rendered frame with an empty `World`, the dev server serves it, and typecheck passes.

## 4. Scaffold the chosen systems

For each system on the build sheet, load its reference file and build it:

| Build-sheet item | Load |
| --- | --- |
| Models, textures, fonts, the loading gate, per-scene asset opt-in, code-split chunks | `references/assets.md` |
| Batched or instanced rendering, shader injection, in-scene text, postprocessing | `references/rendering.md` |
| SFX, music, soundscape, buses, mixing | `references/audio.md` |
| Gravity, collision, support surfaces, slow motion, a physics engine | `references/physics.md` |
| Tweakpane panels, `?debug` flags, perf HUD, capture mode | `references/debug-tooling.md` |
| Frame budget, DPR cap, sim windows, pooling, culling | `references/performance.md` |

Build each system as a **system**: its own directory under `src/experience/world/<name>/`, an exported `<NAME>_ENABLED` const in its own file, conditional construction in `World`, and optional chaining (`this.system?.update()`) at every call site. A system the user did not ask for is absent, not stubbed.

Done when: every build-sheet system exists, is constructed by `World` behind its flag, ticks, and disposes.

## 5. Prove the seams

The starter's whole claim is that systems plug in and out. Verify it rather than asserting it:

- Flip each system's `ENABLED` flag to `false`. The app still boots and renders.
- Run the teardown path (re-init, or a `destroy()` call from the console). No console errors, no growing GPU memory across three cycles.
- Confirm nothing outside a system's own directory imports its internals.

Done when: all three checks pass with output you actually looked at.

## 6. Write the docs

Write a `CLAUDE.md` for the new project so the next agent inherits the architecture instead of rediscovering it. Invoke the `bootstrap-claude-md` skill in progressive-disclosure mode — the systems you just built are its `context/*.md` topics.

Done when: `CLAUDE.md` plus a `context/` file per system exists, and each names the files it covers.

## 7. Report

State: the file tree created, the systems built, the systems left out and why, the verification output from step 5, and the first three things the user should build next in this scaffold.
