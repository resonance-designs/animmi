# Anisynth

Version: 0.1.0

Vector wavetable playground built with Vue 3, Vite, Vuetify, Tone.js, and NexusUI. Four wavetable oscillators are mixed via the classic joystick-style vector blending from the Korg Wavestation, with live Nexus piano input and a 10-lane, 8-page step sequencer. Per-slot wavetable LFOs, vector LFOs, filter choices, and envelopes round out the signal path.

## Features

- Four-slot wavetable vector synth: load wav files from `src/assets/wavetables/**`, blend via X/Y joystick, and LFO-scan each slot’s frames.
- NexusUI controls: piano keyboard, joystick, and 10-row sequencer (16 steps × 8 pages) with per-step note assignment and lane-level mono behavior.
- LFOs on vector position (X/Y) and on wavetable slot frames with adjustable rate/amount.
- Filters: selectable Moog-style low-pass (cutoff/resonance/drive) or comb filter (delay/resonance/dampening), plus filter envelope.
- Envelopes: dedicated amp envelope and filter envelope (attack/decay/sustain/release, plus filter amount).
- Transport-aware: BPM control, start/stop, page-aware loop length (only loops through pages containing active steps).

## Requirements

- Node.js 18+ and npm

## Setup

```bash
npm install
npm run dev
# open the printed local dev URL
```

## Controls

- **Wavetable slots (A–D):** choose wav files; set per-slot LFO rate/amount to scan frames.
- **Vector joystick:** blends A/B/C/D; vector LFOs offset around the current joystick position and animate the pad marker.
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
- Joystick, piano, and sequencer are NexusUI widgets; if the pad marker isn’t moving, ensure the vector LFO rates/amounts are non-zero. The vector LFO loop updates the Nexus position marker directly in `src/App.vue`.

## Troubleshooting

- No sound: ensure audio is unlocked (click any UI control) and `npm install` has run; check master gain/limiter not muted.
- Joystick doesn’t animate: set non-zero vector LFO rate/amount; Nexus pad updates in the animation loop in `src/App.vue`.
- Sequencer artifacts: verify slot wavetables are loaded; per-lane mono and pooled voices minimize clicks.
- Comb filter quiet/harsh: adjust delay/resonance/dampening; switch to Moog filter to compare.
- High CPU: reduce frame count per wavetable (fewer frames), lower LFO rates, or shorten sequencer length/pages.

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
