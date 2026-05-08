# YUCCA-FX 8-BIT

A chiptune-flavored sound effect synthesizer that runs entirely in the browser. Built for game developers, chiptune musicians, and anyone who wants to crack out NES-style blips, lasers, explosions, and weird textural noise without installing a single thing.

**Live:** [yuccabucca.com/yucca-fx](https://yuccabucca.com/yucca-fx)
**Source:** single-file vanilla JS, no build step, no dependencies.

![NES palette · Press Start 2P · CRT scanlines](https://img.shields.io/badge/aesthetic-NES%20fantasy%20cartridge-ffec27?style=flat-square&labelColor=06040f)
![Web Audio](https://img.shields.io/badge/audio-Web%20Audio%20API-3df56b?style=flat-square&labelColor=06040f)
![No build](https://img.shields.io/badge/build-none-3ad7ff?style=flat-square&labelColor=06040f)

---

## What it does

YUCCA-FX is a complete 8-bit sound design environment. You get a synthesizer with the right voicing for retro game audio (Fourier-synthesized pulses with NES duty cycles, 4-bit-stepped triangle, real LFSR noise with NES tap modes), wrapped in a layered effects chain and a euclidean rhythm sequencer that can record itself to a file.

Every sound you make can be exported individually as WAV or MP3, queued up in batches, or rendered as a multi-cycle sequencer loop. You can save presets, share patches via URL, and your work persists across browser sessions automatically.

It's free, open source, runs offline once loaded, and doesn't track you.

---

## Features

### Synthesis engine
- **Authentic chiptune voicing.** Pulse waves with selectable duty (12.5 / 25 / 50 / 75 percent), 4-bit-stepped triangle via `PeriodicWave`, real LFSR noise with both long and short (metallic) NES tap modes
- **Draggable ADSR envelope** with a live canvas display — grab any of the four knee points (Attack / Decay / Sustain / Release) directly with the mouse or touch
- **Filter** with realtime frequency response visualization that updates as you move sliders, plus an envelope-modulated cutoff for sweeps
- **LFO** with selectable waveform routed to pitch, filter, or amplitude
- **Pitch envelope** for sweeps, drops, and impacts
- **Live waveform scope** showing the actual rendered output

### Effects chain
- **Delay** with feedback and a safety FB FADE that prevents runaway feedback from blowing up your ears
- **Reverb** built from a 4-bit-quantized noise impulse, lowpass-filtered for retro warmth
- **RetroVerb** — short slapback delay tuned for NES-style space (separate from the longer modulation delay)
- **Drive** waveshaping for grit and saturation

### Destroy section
- **Circuit Bent** — scheduled random parameter pokes (filter cutoff, pitch, amplitude, resonance, sweeps) that glitch the sound mid-tone. Tunable rate and intensity. Always volume-safe — bent events never push past the existing safety limits
- **Mutant Delay** — instead of a normal delay, this triggers full mutated copies of the patch at configurable intervals. Each copy can shift pitch, filter, duty cycle, drive, and oscillator type. Includes NO NOISE / ONLY NOISE constraints for tight control over the cascade

### Lissajous Sequencer
- Three independent **euclidean rhythm tracks** (pulses-in-steps with even distribution)
- **Pitch offset** per track (-12 to +12 semitones)
- **Lissajous XY visualizer** flashes the color of whichever track just fired
- **Variation engine** with deterministic per-step mutations — same pattern, same variations, so loops groove instead of flickering randomly
- **Sequence recording** — render N cycles to WAV / MP3 with automatic LCM-based loop length detection (cleanly looping output)

### Export workflow
- Single-patch export to **WAV or MP3**
- **Export Queue** holds up to 20 patches with editable names
- **Batch export** as separate files or bundled into a **single zip** (custom pure-JS PKZIP implementation, no library needed)
- **Batch rename** with auto-numbered padding (`coin_01`, `coin_02`, ...)
- Per-export options: **mono / stereo**, **normalize to -6 dBFS**, **brick-wall limiter** at -1 dBFS (on by default)

### Quality of life
- **Master volume slider + VU meter + clip LED** in the scope bar — performant single-rAF loop, GPU-composited fill, throttled DOM writes
- **+Q quick-queue button** on the scope bar — single tap to add the current patch to the export queue without opening the menu
- **BACK button** — undo your way back through preset loads, randomize, init, and factory loads (32-deep history)
- **Persistent state** — your current patch and export queue survive browser restarts via localStorage
- **Tooltips** — hover or focus any control for a one-line explanation in the status bar (39 tooltips covering the most-used controls)
- **Sequencer help** — a `?` button in the transport row opens a structured guide
- **Confirmation modals** for destructive actions, with don't-show-again checkboxes
- **URL sharing** — every patch encodes to a base64 hash you can share as a link

---

## Hotkeys

| Key | Action |
|-----|--------|
| `SPACE` | Play current patch |
| `R` | Randomize patch (with confirmation, suppressible per session) |

---

## Browser support

Modern Chromium-based browsers and Firefox are first-class. Safari works but ADSR canvas dragging may behave slightly differently on iOS. Requires Web Audio API, Pointer Events API, and ES2020 syntax. No polyfills shipped.

---

## Running locally

It's a single HTML file. Open it in a browser. That's it.

```bash
# Optional: serve locally so localStorage uses a stable origin
python3 -m http.server 8000
# then visit http://localhost:8000/yucca-fx-8bit.html
```

For deployment to a static host (GoDaddy, Netlify, GitHub Pages, anywhere): just upload the file. No build pipeline, no environment variables, no backend.

---

## Architecture notes

If you're poking at the source, the file is structured roughly:

1. **Color palette + initial state** — `state` object holds patch, sequencer, export options, and runtime UI references
2. **Patch shape** — `initPatch()` defines the canonical structure; `setPatch()` deep-merges with defaults so old presets without newer fields don't crash
3. **Audio engine** — `buildSound(ctx, patch, dest, gain, startTime)` is the main sound builder, used identically for live preview and offline rendering
4. **Effects routing** — main hit goes through filter → ampGain → fxIn → drive (waveshaper) → out → dest, with parallel sends for delay / reverb / retroverb
5. **Sequencer** — `startSequencer()` / `seqTick()` schedule via `setTimeout` with a recomputed step time per tick
6. **Offline rendering** — `renderToBuffer(p)` and `renderSequencer(cycles)` use `OfflineAudioContext`, optionally piped through `createLimiterChain(ctx)`
7. **UI** — `build()` constructs everything from helper components: `makeSection`, `makeSlider`, `makeBoolPill`, `makePixelButton`, `makeDropdown`. Every patch path is reactive via the `subscribe` / `notifyLive` system
8. **Persistence** — `loadPresets` / `loadQueue` / `loadAutosave` on boot; `savePresets` / `saveQueue` / `scheduleAutosave` (debounced) on mutation

State paths use dot notation (`amp.attack`, `mutantDelay.pitchShift`). Sliders bind to a path string and read/write via `getPath` / `setPath`.

---

## Contributing

Issues and PRs welcome. Keep the single-file structure intact (no build step). When adding a new patch field, also add it to `initPatch()` so backwards-compatible deep-merge keeps old presets working.

---

## Credits

Built by [yuccabuccA](https://yuccabucca.com).

Synthesis vocabulary borrowed from countless chiptune musicians and game audio designers who made the NES sound the way it does. MP3 encoding via [lamejs](https://github.com/zhuker/lamejs) (CDN-loaded on demand, only when MP3 export is used).

---

## License

MIT.
