# Physics, collision, and time control

## Pick the model first

| Need | Use |
| --- | --- |
| Scripted movement, gravity, "am I standing on something", "did I touch a hazard" | **Kinematic**: hand-integrated gravity + AABB checks. No engine. |
| Stacking, tumbling, joints, ragdolls, emergent piles | **Rapier** (`@dimforge/rapier3d-compat`) |
| Rolling/driving vehicles, dozens of colliding bodies | Rapier |

Kinematic covers more games than people expect, and it is worth defending: it costs no wasm payload, is frame-deterministic, and — decisively — lets animation own the motion. A jump that must land exactly on a target is a tween; asking a solver for it is a fight. Reach for an engine when the *emergent* behaviour is the point, not when you merely have gravity.

If the answer is Rapier, still keep the interface below (`update()` / `dispose()`, `Experience.get().time.delta`, one fixed step accumulator) so it stays a swappable **system**.

## Kinematic character

Split the character across small collaborators rather than one god class — each is testable and readable alone:

| File | Owns |
| --- | --- |
| `character.ts` | assembly, mesh, `update()` order |
| `character-input.ts` | raw input → intents |
| `character-state.ts` | the FSM, dispatches `STATE_CHANGED` |
| `character-controller.ts` | intents + state → actions |
| `character-physics.ts` | gravity, velocity integration, knockback |
| `character-collision.ts` | bounds, hazard/water tests, death cause |
| `character-support-state.ts` | what it is standing on, and whether that thing still holds |
| `character-animation.ts` | tweens and rig poses |

### Gravity

```ts
const GRAVITY = 9.81 * 6;              // stylized games want heavier-than-real
const KNOCKBACK_DAMPING = 0.55;        // per-second surviving fraction of lateral drift

setGravityActive(active: boolean, initialVelocityY = 0) {
  if (this.gravityActivated === active) return;
  this.gravityActivated = active;
  if (!active) { this.velocityY = this.velocityX = this.velocityZ = 0; return; }
  // Honor pre-existing downward momentum: a jump arc's terminal velocity must
  // carry into the fall, or landing on a surface that gives way reads as the
  // character briefly catching on it.
  this.velocityY = Math.min(this.velocityY, initialVelocityY);
}
```

Keep lateral velocity at zero for normal movement (tweens own it) and non-zero only while a knockback plays out — then `update()` skips the lateral integration entirely on almost every frame.

### Collision

Refresh AABBs on `time.stepTick` (~30 Hz), not every frame — see `core-runtime.md`. Cache the mesh's local bounds once and transform them, rather than recomputing from geometry.

Model a **support surface** as an interface rather than special-casing each thing that can be stood on:

```ts
interface SupportSurface {
  /** World-space top plane the character rests on, or null when it cannot hold. */
  getSupportY(x: number, z: number): number | null;
  readonly supportBounds: THREE.Box3;
}
```

A moving platform, a door that opens, a bridge that breaks, and solid ground all implement it. The character asks its current support each frame whether it still holds; `null` starts a fall. This is what keeps "the ground moved" and "the ground vanished" from being two different code paths.

### Death and knockback

Distinguish the **cause** (`"hazard" | "water" | "fall"`) at the collision site and carry it in the event — reactions, sounds, and camera all branch on it. Knockback is an upward pop plus lateral drift that decays; the ordinary fall/water collision then finishes the job, so there is no separate "knocked off" death path.

For a fall that should pay off with a specific reaction, let whatever *started* the fall **arm** a flag (`armLandingReaction("heavy")`) and let the collision that ends the fall **consume** it. The impact site does not know the fall was a scripted failure; the thing that pushed does. Clear the flag if a different cause ends the fall first.

## Slow motion

Global time control has to move two clocks together. Anything that ticks from `Time` and anything animating on GSAP's global timeline must slow in lockstep or the world visibly tears.

```ts
class SlowMotion {
  trigger(scale: number, holdDuration: number, rampDuration: number) { /* … */ }

  update() { /* advance the envelope in GAME seconds */ }

  reset() { this.phase = "idle"; this.applyScale(1); }

  private applyScale(scale: number) {
    this.time.timeScale = scale;
    gsap.globalTimeline.timeScale(scale);
    setGlobalAudioRate(scale);         // the mix pitch-dives with the world
  }
}
```

Details that matter:

- **Track the envelope in game seconds** (the scaled `time.delta`), so a trigger sized to an animation's duration stays synced with that animation however deep the slowdown.
- **Floor `scale` above zero.** At 0, game time stops advancing and the envelope can never end.
- **Carry the hold's overshoot into the ramp** so total length is exact.
- **Linear ramp in game time reads as an ease-out in wall-clock** — it creeps off the slow scale, then accelerates into full speed. That is usually the feel you want.
- **Reset on teardown.** GSAP's global timeline outlives the `Experience`; a destroy mid-slow-motion otherwise leaves the whole page slowed.

It ticks from `Experience.tick()` immediately after `time.update()`, before anything reads `delta` — the slot shown in `core-runtime.md`'s frame.

## Debug visualization

Behind `?debug`, draw what the physics believes: `Box3Helper` on the character bounds, a wireframe of every support surface's plane and bounds, and the current state name. A support bug is invisible in the render and obvious the moment you draw the planes. Keep the helpers in one debug system with its own `ENABLED` gate so production never allocates them.
