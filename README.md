# Anisynth

Version: 0.1.1

Vector wavetable playground built with Vue 3, Vite, Vuetify, Tone.js, and NexusUI. Four wavetable oscillators are mixed via the classic joystick-style vector blending from the Korg Wavestation, with live Nexus piano input and a 10-lane, 8-page step sequencer. Per-slot wavetable LFOs, vector LFOs, filter choices, and envelopes round out the signal path.

## Features

- Four-slot wavetable vector synth: load wav files from `src/assets/wavetables/**`, blend via X/Y joystick, and LFO-scan each slot’s frames.
- NexusUI controls: piano keyboard, joystick, and 10-row sequencer (16 steps × 8 pages) with per-step note assignment and lane-level mono behavior.
- LFOs on vector position (X/Y) and on wavetable slot frames with adjustable rate/amount, selectable shapes (sine/triangle/square/saw/S&H), on/off switches, and optional sync-to-steps so a cycle can align to 2/4/16 sequencer steps.
- Filters: selectable Moog-style low-pass (cutoff/resonance/drive) or comb filter (delay/resonance/dampening), plus filter envelope.
- Envelopes: dedicated amp envelope and filter envelope (attack/decay/sustain/release, plus filter amount).
- Transport-aware: BPM control, start/stop, page-aware loop length (only loops through pages containing active steps). Transport start/stop also starts/stops all LFOs.

## Requirements

- Node.js 18+ and npm

## Setup

```bash
npm install
npm run dev
# open the printed local dev URL
```

## Controls

- **Wavetable slots (A-D):** choose wav files; LFOs per slot have shape (sine/triangle/square/saw/S&H), on/off, optional sync to sequencer steps (steps per cycle), rate (Hz when unsynced), and amount (frame scan depth).
- **Vector joystick:** blends A/B/C/D; vector LFOs offset around the current joystick position and animate the pad marker. Same controls as slot LFOs (shape/on/off/sync or Hz, amount).
- **Sequencer:** 10 lanes (polyphony per row), 16 steps × 8 pages. Page select controls the visible page; playback loops through pages containing active steps. Assign notes per step via the note selector.
- **Filters:** pick Moog or Comb. Moog exposes cutoff/resonance/drive; Comb exposes delay/resonance/dampening.
- **Envelopes:** amp ADSR; filter ADSR plus amount. Filter envelope modulates Moog cutoff or Comb dampening.
- **Global gain:** left card slider controls master output gain; master limiter is applied downstream.

## Scripts

- `npm run dev` – start the Vite dev server.
- `npm run build` – production build.
- `npm run preview` – preview the production build.

## Notes

- Wavetables are decoded from the wav files in `src/assets/wavetables/**` by slicing multiple cycles into frames and converting them to partials for custom oscillators.
- Joystick, piano, and sequencer are NexusUI widgets; if the pad marker isn't moving, ensure the vector LFO rates/amounts are non-zero. The vector LFO loop updates the Nexus position marker directly in `src/App.vue`.
- LFO Sample & Hold uses Tone’s random behavior while the frame-morphing code holds the random values per step; slot and vector LFOs share the same shape/on/off/sync controls.

## Troubleshooting

- No sound: ensure audio is unlocked (click any UI control) and `npm install` has run; check master gain/limiter not muted.
- Joystick doesn’t animate: set non-zero vector LFO rate/amount; Nexus pad updates in the animation loop in `src/App.vue`.
- Sequencer artifacts: verify slot wavetables are loaded; per-lane mono and pooled voices minimize clicks.
- Comb filter quiet/harsh: adjust delay/resonance/dampening; switch to Moog filter to compare.
- High CPU/crackle when moving mouse: production build is lighter; also reduce LFO rates/amounts and use fewer steps/pages. Tone context `lookAhead`/`updateInterval` are tuned in `ensureAudio` to reduce UI-induced crackles.

## Design Notes

- Voice pool: oscillators and filters are pooled to avoid per-step allocations; gains retune per note.
- Wavetable frames: wav files are sliced into multiple cycles -> partial frames; per-slot LFO scans frames.
- Vector morph: joystick + vector LFOs drive mix weights; Nexus UI reflects the animated position.
- Filters/envelopes: Moog (LPF emulation) or comb filter with amp/filter envelopes modulating cutoff/dampening.

## Future Roadmap

- Preset system for slots/LFOs/filters/envelopes/sequencer pages.
- Per-lane velocity/probability and swing/shuffle.
- Additional filter models and per-slot filters.
- Global FX (reverb/delay) with send levels per lane.
- Export/import sequencer patterns and MIDI learn for key controls.
