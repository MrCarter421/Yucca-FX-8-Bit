# CHANGELOG

All notable changes to YUCCA-FX 8-BIT.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project adheres to semantic-ish versioning (minor versions add features, the patch number would track bugfix-only releases).

---

## [1.5] — Bulletproofing release

### Added
- **Tooltip system.** New `tip(element, text)` helper that wires both the native `title` attribute (for browser hover tooltips and accessibility via `aria-label`) and a custom `.status-tip` line in the status bar that responds immediately to hover, focus, or touch. 39 tooltips wired across the most-used controls — every header button, scope `+Q`, master volume, all section ON/OFF toggles, FB FADE pills, NO NOISE / ONLY NOISE pills, sequencer transport, recording controls, queue actions, batch / zip exports, export option pills, RESEED, SAVE, and preset name input
- **BACK button** with 32-deep patch history. Undoes preset loads, factory loads, randomize, init, and URL-shared overrides. Visually fades when the history stack is empty. Half-size styling so it sits between PLAY and RANDOM without competing for attention
- **Persistent export queue.** Saves to localStorage on every mutation (add, remove, clear, batch rename, batch operations) and restores on boot. Survives browser restarts
- **Persistent current patch.** Debounced autosave (400ms) captures both the patch and the full sequencer state (tracks, BPM, variation engine settings) on every parameter change. Restoration priority on boot: URL hash > autosave > default INIT
- **Header reorganization.** `PLAY · | · BACK · RANDOM · | · PRESETS · EXPORT · INIT · SHARE` with visible dividers between functional clusters. FACTORY dropdown removed entirely — factory sounds now appear at the top of the PRESETS popup, above the user PRESET BANK
- **`half: true` modifier on `makePixelButton`** for slim secondary buttons. Used by BACK
- **`back` icon SVG** added to the ICONS object

### Changed
- R hotkey now routes through the same `showConfirm` flow as the RANDOM button, sharing the same suppression key. Once you opt out of the confirmation popup for the session, the hotkey works as a one-tap shortcut
- `setPatch()` accepts an optional `{ skipHistory: true }` flag for restorations that shouldn't pollute the history stack (BACK itself, URL-hash boot, autosave restore)

### Fixed
- Quick-queue counter (`+Q [n]` badge on the scope bar) now updates correctly when the queue is cleared from the EXPORT menu. Earlier versions had an early-return ordering bug that skipped the badge sync on the empty-state path

---

## [1.4] — Quick-queue + sequencer help

### Added
- **`+Q` quick-queue button** in the master scope bar. One tap adds the current patch to the export queue without opening the EXPORT menu. Live counter badge shows current queue size. Border lights up green when items present, red when at the 20-item cap
- **`?` help button** in the sequencer transport row. Opens a structured guide modal covering tracks, pulses, steps, pitch offset, the Lissajous visualizer, the variation engine, recording loops, and tips
- **`showInfoModal({ title, bodyHtml, accent })` helper** — single-button informational modal with rich HTML body support (styled `h4`, `code`, `strong`, `em`). Foundation for any future help screens

### Changed
- Both the EXPORT popup's ADD CURRENT button and the new scope `+Q` button now route through a shared global `addCurrentToQueue()` helper, keeping the two UIs in sync via a `state._queueRefresh` callback

---

## [1.3] — Limiter, preset bundling, button readability

### Added
- **Brick-wall limiter** in export options, on by default. Implemented as a `DynamicsCompressorNode` with limiter settings (-1 dBFS threshold, hard knee, 20:1 ratio, 1ms attack, 50ms release). Wired into both `renderToBuffer` and `renderSequencer`. Prevents clipping in exported files
- **Sequencer state in saved presets.** SAVE now writes both the patch and a `snapshotSequencer()` blob (BPM, tracks, variation engine). LOAD restores both. Old presets without the sequencer field still load the patch alone with no errors
- **Auto-contrast button text.** `makePixelButton` now picks light or dark text based on the perceived brightness of the accent color (ITU-R BT.601 luma weighting). INIT and KEEP IT buttons (dark `bg3` accent) are now clearly readable. Yellow / green buttons keep their dark text. Universal — applies to every existing and future button

### Fixed
- **Circuit Bent now exports correctly.** Earlier versions silently dropped circuit-bent events in offline rendering because `setValueAtTime` calls were conflicting with in-progress ADSR ramps (Web Audio gives undefined behavior for that combo, and offline contexts let the ramp win). Fixed by gating event scheduling to start after `attack + decay` completes, where the ADSR has landed at sustain (a static value) and `setValueAtTime` works reliably. Audible effect is the same, just no longer silently eaten

---

## [1.2] — Export Queue + Sequencer Recording

### Added
- **Export Queue** capped at 20 items, in the EXPORT popup
  - `+ ADD CURRENT` clones the current patch into the queue
  - Editable name input per item, live commit
  - `BATCH RENAME` with auto-numbered padding (`coin_01`, `coin_02`, ...)
  - `BATCH WAV` / `BATCH MP3` for separate-file downloads (250ms staggered to avoid browser multi-download blocking)
  - `ZIP (WAV)` / `ZIP (MP3)` for bundled `.zip` downloads
  - `CLEAR` button with confirmation
  - Live progress display: `RENDERING 3 OF 12 · coin_03` with progress bar
- **Custom ZIP builder** (no library). Pure-JS PKZIP with STORE compression and proper CRC-32. Tested round-trip through system `unzip` with text and binary payloads. STORE compression chosen because MP3 is already compressed and WAV doesn't deflate meaningfully without significant bundle-size cost
- **Sequencer Recording** in the sequencer transport row
  - `CYCLES` slider 1–8 (default 4)
  - `REC WAV` / `REC MP3` buttons
  - Live info display: `16 STEPS × 4 = 8.00s`, updates as tracks toggle, BPM changes, or cycles change
  - LCM-based loop length detection across active tracks for clean looping. Capped at 64 sixteenths to prevent absurd LCMs from coprime track counts. Hard 30-second render cap as final safety
  - Render walks the same engine as the live sequencer (euclidean rhythm + variation + full mutant cascade) so recordings match what you hear

---

## [1.1] — Improved Randomization + Master Volume + VU Meter

### Added
- **Master volume slider** in the scope bar. Routes all live playback through a persistent master gain node so changes are heard immediately even on currently-playing sounds. Doesn't affect exports
- **VU meter + clip LED** with peak hold. Single shared rAF loop, pre-allocated `Uint8Array(512)` reused every frame, GPU-composited fill (`will-change: width`), DOM writes throttled to changes ≥0.3% so the meter is essentially free when idle. Clip LED catches any sample reaching ≥0.99 amplitude with a 220ms flash hold

### Changed
- **Randomization rewritten with two-layer defense against silent patches.**
  1. Constrained ranges in `randomPatchAttempt()` couple filter type and cutoff with the chosen note. Lowpass cutoff is at minimum `noteHz × 1.5` (and ≥400Hz) so the fundamental always passes. Highpass cutoff is capped at `noteHz × 0.8` so it can't strip tonal content. Bandpass center is bound to the note's frequency. Pitch envelope amount narrowed (±36 → ±24), sustain biased away from zero unless attack+decay together exceed 80ms, release floor raised (50→80ms), duration floor raised (80→120ms)
  2. New `estimateAudibility(p)` heuristic scores patches 0..1 by checking for known silence patterns. `randomPatch()` retries up to 8 times, accepting the first patch scoring ≥ 0.6, falling back to the best of the lot. Per-call cost ~6μs. In practice the constrained ranges produce passing patches on the first try 100% of the time across 1000-iteration tests, with the heuristic catching pathological edge cases

---

## [1.0] — Initial release

The base build, including:

- Authentic chiptune voicing (Fourier-synthesized pulse, 4-bit triangle, real LFSR noise with NES tap modes)
- Draggable ADSR canvas with live knee-point dragging
- Filter with realtime frequency-response visualization
- LFO routed to pitch / filter / amp
- Pitch envelope
- Live waveform scope using `OfflineAudioContext` rendering
- Delay with safety FB FADE
- Reverb (4-bit-quantized noise impulse, lowpass-filtered)
- RetroVerb (NES-style slapback with tone control)
- Drive waveshaping
- Circuit Bent (random parameter pokes with safety clamps)
- Mutant Delay (full mutated patch copies, NO NOISE / ONLY NOISE constraints)
- Three-track Lissajous euclidean sequencer with variation engine
- Lissajous XY visualizer with track-color flashes
- Single-patch export to WAV / MP3 (lamejs CDN-loaded on demand)
- Factory presets (coin / laser / explode / etc)
- User preset bank (24 max, oldest drops, localStorage)
- URL share (base64-encoded patches in the hash)
- Confirmation modals with don't-show-again checkboxes
- Hotkeys (SPACE = play, R = randomize)
- NES palette + Press Start 2P + VT323 fonts + CRT scanlines + vignette overlay
