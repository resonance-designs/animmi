# Changelog

All notable changes to this project will be documented here.

## [0.1.0] - 2026-01-02

- Vue 3 + Vite scaffold with Vuetify UI.
- Four-slot vector wavetable synth using Tone.js custom oscillators and NexusUI joystick.
- Wavetable frame scanning with per-slot LFO rate/amount controls.
- NexusUI piano input and 10-lane, 16-step × 8-page sequencer with per-step note assignment and lane-level mono behavior.
- Vector LFOs (X/Y) that animate the joystick and modulate mix weights.
- Filter section with Moog-style low-pass (cutoff/resonance/drive) or comb filter (delay/resonance/dampening).
- Amp envelope and filter envelope (ADSR with amount).
- Page-aware looping (only loops through pages containing active steps); transport start/stop and BPM control.
- Wavetable assets under `src/assets/wavetables/**`; master limiter and output gain control.
