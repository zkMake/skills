---
name: optimize-audio
description: Shrink audio files (mp3, wav, ogg, m4a, flac) by re-encoding with ffmpeg at lower bitrates / fewer channels. Use when the user wants to reduce audio file size, compress audio, optimize mp3/wav/etc for the web, or mentions an audio file being too large.
---

# optimize-audio

Re-encode audio via ffmpeg to cut file size. Offer a few bitrate/channel options, preview them, then replace the original with the user's pick.

## Preflight: ffmpeg

```bash
command -v ffmpeg >/dev/null || echo "MISSING"
```

If missing, stop and tell the user:

> ffmpeg is required. Install it:
> - macOS: `brew install ffmpeg`
> - Ubuntu/Debian: `sudo apt install ffmpeg`
> - Windows: `winget install ffmpeg` or https://ffmpeg.org/download.html
>
> Let me know once it's installed.

Do not proceed until they confirm.

## Workflow

1. **Inspect the source** — report current size + codec + bitrate + channels + duration:
   ```bash
   ls -lh <file>
   ffprobe -v error -show_entries stream=codec_name,sample_rate,channels,bit_rate -show_entries format=duration,bit_rate <file>
   ```

2. **Generate 3–4 options into `/tmp/<basename>-test/`** spanning quality tiers. Pick defaults from the source:
   - Current bitrate is a ceiling — don't re-encode higher.
   - Skip options that would make the file larger.
   - Always include one "sweet spot" (~60% size reduction, near-transparent).

3. **Present a comparison table** with size, % savings, and a one-line tradeoff per option. Recommend one explicitly. Give the user the `/tmp/...` paths and tell them macOS Finder hides `/tmp` (Cmd+Shift+G to navigate, or `open /tmp/<basename>-test/`).

4. **Replace on approval** — copy the chosen preview over the original. Use an absolute path for `cp` (shell cwd can drift between Bash calls).

## Option presets

| Use case | Preset | Typical savings vs 256k stereo |
|---|---|---|
| Ambient / background music loop | 96k stereo mp3 | ~60% |
| Speech / narration | 64k mono mp3 | ~75% |
| Short SFX (<2s) | 96k mono mp3 | — |
| Need smallest with good quality | 64k opus (`.ogg`) | ~70% (check player/browser support first) |
| Transparent fallback | 128k stereo mp3 | ~50% |

## ffmpeg commands

```bash
# mp3, N kbps, stereo (N = 64/96/128/192)
ffmpeg -y -i in.mp3 -b:a 96k -ac 2 out.mp3

# mp3, N kbps, mono
ffmpeg -y -i in.mp3 -b:a 64k -ac 1 out.mp3

# opus in ogg container (best quality/size, but check codec support)
ffmpeg -y -i in.mp3 -c:a libopus -b:a 64k out.ogg

# aac in m4a (good for Safari-first pipelines)
ffmpeg -y -i in.mp3 -c:a aac -b:a 96k out.m4a
```

Run all encodes in parallel within one Bash call (`&&` chain) to save time.

## Notes

- Opus/ogg: verify the app's audio library supports it before recommending (Howler.js does, but needs a format hint on Safari).
- Check the repo for an audio registry (e.g. `audio.ts`) — if the file extension changes, the import path changes too.
- Don't `git add` until the user confirms the preview sounds acceptable.
