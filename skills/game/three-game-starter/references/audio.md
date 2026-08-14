# Audio

Three layers, one mixer, one place where sounds are constructed. Built on Howler (`html5` streaming for music, buffered for everything else).

## The three layers

| Layer | What it is | Bus |
| --- | --- | --- |
| **Music** | Set-and-forget tracks under the game. One track, or a shuffled rotation. | `music` |
| **Soundscape** | Per-scene ambience, assembled live from a looping bed plus randomized one-shot emitters. | `ambience` |
| **SFX** | One-shots tied to a moment: a jump, an impact, a pickup, a death. | `sfx` |

Keeping them separate is what lets an outro fade the ambience to silence without dimming the SFX, or duck the music without touching anything else.

## One module owns every Howl

Every `new Howl` lives in `core/audio.ts`; no other file constructs one. Howls are created lazily on first access and survive re-inits (they are expensive to rebuild) — but bus levels and the live-instance map do not.

Public surface:

```ts
playSfx(name: SfxName, onPlay?: (soundId: number, howl: Howl) => void): number
playSfxLoop(name, { volume, fadeInMs? }): number
stopSfxLoop(soundId, fadeOutMs?): void
getSfx(name): Howl                       // only when you need .fade()/.volume() directly
playBackgroundMusic(): void
playSoundscapeBed(sample, { volume, fadeInMs }): number
playSoundscapeEmitter(sample, { volume, pan, rate, onEnd? }): number
stopSoundscapeBed(soundId, fadeOutMs): void
getBusVolume(bus) / setBusVolume(bus, value, fadeMs?)
stopAllOnBus(bus) / countActiveOnBus(bus)
setInstanceBaseline(soundId, baseline, fadeMs?)
setGlobalAudioRate(rate): void
registerAudioWithManager(manager, { sfx, soundscapeSamples, warmMusic }): void
resetAudioState(): void
```

## The bus formula

Final volume is always `master × its-bus × its-own-baseline`. Call sites declare a **baseline**; nothing outside the audio module touches a Howl's volume.

Track every playing instance in a map keyed by sound id with its bus and baseline. That single map is what makes three things work: a `setBusVolume` re-scales sounds already playing, a scene fades its ambience as one unit, and teardown stops everything.

Two traps:

- **A loop must not go through `playSfx`.** `playSfx` untracks the instance on its first `end` event — correct for a one-shot, fatal for a loop, which then stops answering to bus changes, time scale, and `resetAudioState`. Use the `playSfxLoop` / `stopSfxLoop` pair, and give the caller ownership: ride its level with `setInstanceBaseline`, and always stop it in `dispose()`.
- **A distance-attenuated cue needs its baseline set inside the play callback**, not after. Pass the `onPlay` callback, set the per-instance baseline there, and re-apply the bus formula afterward so a later bus change still rescales the live instance.

## Soundscape

A soundscape is a **recipe**, not a track — new ones need data, not code:

```ts
type SoundscapeDef = {
  bed: { sample: SampleName; volume: number };
  emitters: {
    samples: SampleName[];      // picked at random per firing
    intervalS: [number, number];
    volume: [number, number];
    pan: [number, number];
    rate: [number, number];
  }[];
  maxConcurrentEmitters: number;
};
```

The bed loops for the whole scene and is the floor of the ambience. Each emitter schedules itself at a random interval inside its window and randomizes volume, pan, and pitch on every firing — that jitter is what stops two seagull samples from sounding looped. Cap concurrent emitter one-shots so a busy stretch cannot pile into noise.

Bake the bed's volume into the recipe, tuned so the bed sits *under* the emitters at the right ratio. Expose a debug "bed gain" multiplier for tuning that ratio (the ambience bus slider moves bed and emitters together and so cannot tell you anything about it), and treat it as a tuning aid: not persisted, resets to 1.

## Music rotation

Define tracks as data with an `enabled` flag, so a track can be benched without deleting the asset and a disabled track is never even loaded:

```ts
const BACKGROUND_MUSIC_TRACK_DEFS = [{ id: "bg-01", src: bg01, enabled: true }, /* … */];
const BACKGROUND_MUSIC_TRACKS = BACKGROUND_MUSIC_TRACK_DEFS.filter((t) => t.enabled);
```

Rotation contract: random first track, a full shuffled pass over every track before any repeat, never the same track twice in a row. Transition sequentially — fade the outgoing to 0, and on its `fade` completion stop it and fade the incoming in from 0. A brief silent gap is fine; overlapping two tracks is not. Keep the natural `end` event as a hard-cut fallback, and warm the next track's Howl while the current one plays.

Music **streams** (`html5: true`, `preload: "metadata"`), which changes three things:

- The advance scheduler cannot read `duration()` on the `play` event yet — defer and re-arm on the Howl `load` event.
- Defer the incoming track's fade-in to its own `play` event, or buffering eats the ramp.
- Streamed media mutes below 0.5× playback rate in browsers, so floor the global rate there and disable `preservesPitch` so it tape-drops instead of time-stretching.

## Loading

Register only the cues the current scene needs, wired into the same `LoadingManager` as models and textures so the progress bar reflects reality:

```ts
registerAudioWithManager(manager, {
  sfx: Array.from(requirements.sounds),
  soundscapeSamples: Array.from(requirements.soundscapeSamples),
  warmMusic: true,
});
```

`ALWAYS_LOADED_SFX` is the baseline (jump/land/hit/death/UI — whatever every scene uses). Everything else is opted in through the per-entity `ASSET_MANIFEST` and per-soundscape sample manifest (see `assets.md`). `warmMusic` constructs only the *first* track's Howl to kick off its metadata fetch — never gate the loading screen on a streamed track, which will not reach `canplaythrough` until it buffers.

Nothing is audible on page load: browsers require a gesture. Start music and the soundscape on the first real interaction, or at the end of the loading/intro transition.

## Time scale

If the game has slow motion, the mix follows it: `setGlobalAudioRate(scale)` multiplies every live instance's own base rate and every future play, called from the same controller that sets `time.timeScale`. Reset it to 1 in `resetAudioState()`.

## Teardown

`resetAudioState()` — called from `Experience.destroy()` — stops every tracked instance, clears the live-instance map, resets bus volumes to defaults, resets the global rate, and clears the music rotation's started flag and pending advance timer. Without it, a scene that faded its ambience out hands the next scene a silent ambience bus.

## Asset prep

- Check loudness before importing: `ffmpeg -af volumedetect` and confirm `max_volume` is near 0 dB. Generated audio is often near-silent. Trim dead air and peak-normalize.
- Re-encode to `.ogg` (opus) before committing. Howler picks the codec from the extension, so the import path must match the file on disk.
- Adding an SFX is: import the asset, add an entry to `SFX_DEFS`, then add its key to `ALWAYS_LOADED_SFX` **or** to the relevant entry in the asset manifest. Adding a soundscape sample is: drop the file in, add it to the sample defs, list it under its soundscape id in the manifest, and reference it from a recipe.

## Debug panel

Under `?debug`: sliders per bus (persisted to localStorage so a dev mix survives reloads), a soundscape picker that swaps the active recipe live, the bed-gain slider, a "fire now" button per emitter, and a live per-bus count of playing instances refreshed a few times a second. That count is the leak detector — a number that grows and never drops is an untracked loop.
