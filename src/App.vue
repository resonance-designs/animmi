<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue';
import * as Tone from 'tone';
import * as Nexus from 'nexusui';

const NOTE_POOL = [
  'C2', 'D2', 'E2', 'F2', 'G2', 'A2', 'B2',
  'C3', 'D3', 'E3', 'F3', 'G3', 'A3', 'B3',
  'C4', 'D4', 'E4', 'F4', 'G4', 'A4', 'B4',
  'C5'
];

const wavetableFiles = import.meta.glob('./assets/wavetables/**/*.wav', {
  query: '?url',
  import: 'default',
  eager: true
});

const wavetableOptions = computed(() =>
  Object.entries(wavetableFiles).map(([path, url]) => {
    const parts = path.split('/');
    const label = `${parts.at(-2)}/${parts.at(-1)}`;
    return { label, value: url };
  })
);

const selectedTables = reactive([
  { label: 'Slot A', value: wavetableOptions.value[0]?.value || null, frames: null, lfoRate: 0.15, lfoAmount: 1 },
  { label: 'Slot B', value: wavetableOptions.value[1]?.value || null, frames: null, lfoRate: 0.15, lfoAmount: 1 },
  { label: 'Slot C', value: wavetableOptions.value[2]?.value || null, frames: null, lfoRate: 0.15, lfoAmount: 1 },
  { label: 'Slot D', value: wavetableOptions.value[3]?.value || null, frames: null, lfoRate: 0.15, lfoAmount: 1 }
]);

const gainSliderHost = ref(null);
const xyHost = ref(null);
const pianoHost = ref(null);
const sequencerHost = ref(null);

const gainValue = ref(-6);
const isAudioReady = ref(false);
const vectorBase = reactive({ x: 0.5, y: 0.5 });
const vectorPos = reactive({ x: 0.5, y: 0.5 });
const vectorLfo = reactive({ xRate: 0.1, xAmount: 0, yRate: 0.13, yAmount: 0 });
const filterType = ref('moog');
const moogParams = reactive({ cutoff: 1200, resonance: 0.5, drive: 0.5 });
const combParams = reactive({ delayTime: 0.02, resonance: 0.6, dampening: 3000 });
const ampEnv = reactive({ attack: 0.01, decay: 0.08, sustain: 0.82, release: 0.32 });
const filterEnv = reactive({ attack: 0.02, decay: 0.12, sustain: 0.5, release: 0.3, amount: 1200 });
const bufferCache = new Map();

const rows = 10;
const stepsPerPage = 16;
const totalPages = 8;
const totalSteps = stepsPerPage * totalPages;
const page = ref(0);
const bpm = ref(110);
const isSequencerRunning = ref(false);
const currentStep = ref(0);
const selectedCell = reactive({ row: 0, column: 0 });
const selectedNote = ref('C3');

const defaultRowNotes = ['C3', 'D3', 'E3', 'G3', 'A3', 'C4', 'D4', 'E4', 'G4', 'A4'];
const stepGrid = reactive(
  Array.from({ length: rows }, (_, r) =>
    Array.from({ length: totalSteps }, () => ({
      active: false,
      note: defaultRowNotes[r] || 'C3'
    }))
  )
);

let joystick;
let piano;
let sequencer;
let loop;

const limiter = new Tone.Limiter(-1).toDestination();
const masterGain = new Tone.Gain(0.9).connect(limiter);
const POLY_VOICES = 12;

const getActivePageCount = () => {
  let lastActivePage = 0;
  for (let p = 0; p < totalPages; p += 1) {
    const start = p * stepsPerPage;
    const end = start + stepsPerPage;
    const hasNotes = stepGrid.some((row) => row.slice(start, end).some((cell) => cell.active));
    if (hasNotes) {
      lastActivePage = p + 1;
    }
  }
  return Math.max(1, lastActivePage);
};

const activeSteps = computed(() => stepsPerPage * getActivePageCount());

const disposeVoice = (voice) => {
  voice?.slots?.forEach((entry) => {
    entry?.osc?.stop?.();
    entry?.osc?.dispose?.();
    entry?.gain?.dispose?.();
    entry?.lfo?.dispose?.();
    entry?.partialLoop?.dispose?.();
  });
  voice?.filter?.dispose?.();
  voice?.mix?.dispose?.();
  voice?.envelope?.dispose?.();
};

const ensureAudio = async () => {
  if (!isAudioReady.value) {
    await Tone.start();
    isAudioReady.value = true;
    startVectorLfos();
  }
};

const loadBuffer = async (url) => {
  if (!url) return null;
  if (bufferCache.has(url)) return bufferCache.get(url);
  const buffer = await Tone.ToneAudioBuffer.fromUrl(url);
  bufferCache.set(url, buffer);
  return buffer;
};

const findZeroCrossing = (arr, start = 0, minDistance = 64) => {
  const len = arr.length;
  let prev = arr[start];
  for (let i = start + 1; i < len; i += 1) {
    const curr = arr[i];
    if (Math.sign(prev) !== Math.sign(curr) && i - start >= minDistance) {
      return i;
    }
    prev = curr;
  }
  return null;
};

const extractCycle = (channel, startIndex = 0, minDistance = 64, maxDistance = 8192) => {
  const start = findZeroCrossing(channel, startIndex, 0) ?? startIndex;
  const end = findZeroCrossing(channel, start, minDistance);
  if (end && end - start <= maxDistance) {
    return { slice: channel.slice(start, end), next: end };
  }
  const sliceSize = Math.min(channel.length - start, 2048);
  return { slice: channel.slice(start, start + sliceSize), next: start + sliceSize };
};

const cycleToPartials = (cycle, harmonics = 24) => {
  if (!cycle?.length) return [];
  const N = cycle.length;

  // remove DC and window to reduce buzz
  const mean = cycle.reduce((a, b) => a + b, 0) / N;
  const windowed = cycle.map((v, i) => {
    const hann = 0.5 * (1 - Math.cos((2 * Math.PI * i) / (N - 1)));
    return (v - mean) * hann;
  });

  const maxAmp = Math.max(...windowed.map((v) => Math.abs(v))) || 1;
  const norm = windowed.map((v) => v / maxAmp);

  const partials = [];
  for (let k = 1; k <= harmonics; k += 1) {
    let real = 0;
    let imag = 0;
    for (let n = 0; n < N; n += 1) {
      const phase = (2 * Math.PI * k * n) / N;
      real += norm[n] * Math.cos(phase);
      imag -= norm[n] * Math.sin(phase);
    }
    const magnitude = Math.sqrt(real * real + imag * imag) / (N / 2);
    partials.push(magnitude);
  }
  return partials;
};

const loadFramesFromWav = async (url, maxFrames = 8) => {
  const buffer = await loadBuffer(url);
  if (!buffer) return [];
  const data = buffer.toArray();
  const channel = Array.isArray(data) ? data[0] : data;
  if (!channel?.length) return [];

  const frames = [];
  let cursor = 0;
  while (frames.length < maxFrames && cursor < channel.length - 128) {
    const { slice, next } = extractCycle(channel, cursor, 64, 4096);
    if (!slice?.length) break;
    frames.push(cycleToPartials(slice));
    cursor = next;
  }

  return frames.length ? frames : [cycleToPartials(channel.slice(0, 2048))];
};

const rotatePartials = (partials, shift = 1) => {
  if (!partials?.length) return [];
  const len = partials.length;
  const out = new Array(len);
  for (let i = 0; i < len; i += 1) {
    out[i] = partials[(i + shift) % len];
  }
  return out;
};

const startVectorLfos = () => {
  const driveUi = (ts) => {
    const now = ts || performance.now();
    const dt = vectorLfoLast ? (now - vectorLfoLast) / 1000 : 0;
    vectorLfoLast = now;
    const rateX = toNumber(vectorLfo.xRate, 0);
    const rateY = toNumber(vectorLfo.yRate, 0);
    const amtX = toNumber(vectorLfo.xAmount, 0);
    const amtY = toNumber(vectorLfo.yAmount, 0);
    vectorLfoPhase.x += 2 * Math.PI * rateX * dt;
    vectorLfoPhase.y += 2 * Math.PI * rateY * dt;
    const lfoXVal = amtX * Math.sin(vectorLfoPhase.x);
    const lfoYVal = amtY * Math.sin(vectorLfoPhase.y);
    const baseX = toNumber(vectorBase.x, 0.5);
    const baseY = toNumber(vectorBase.y, 0.5);
    const x = Math.min(Math.max(baseX + lfoXVal, 0), 1);
    const y = Math.min(Math.max(baseY + lfoYVal, 0), 1);
    vectorPos.x = x;
    vectorPos.y = y;
    const nx = x;
    const ny = 1 - y;
    if (joystick?.set) {
      joystick.set({ x: nx, y: ny });
    } else if (joystick?.position?.set) {
      joystick.position.set({ x: nx, y: ny });
    }
    if (typeof joystick?.draw === 'function') joystick.draw();
    vectorLfoFrame = requestAnimationFrame(driveUi);
  };
  if (!vectorLfoFrame) {
    vectorLfoFrame = requestAnimationFrame(driveUi);
  }
};

const updateVectorLfos = () => {
  // no-op for math-based UI LFOs; watcher keeps API stable
};

const morphPartials = (a = [], b = [], t = 0) => {
  const len = Math.max(a.length, b.length);
  const out = new Array(len);
  for (let i = 0; i < len; i += 1) {
    const av = a[i] ?? 0;
    const bv = b[i] ?? 0;
    out[i] = av * (1 - t) + bv * t;
  }
  return out;
};

const toNumber = (val, fallback = 0) => {
  const num = Number(val);
  return Number.isFinite(num) ? num : fallback;
};

const ensureSelectedPartials = async () => {
  const promises = selectedTables.map(async (slot) => {
    if (slot.value && !slot.frames) {
      slot.frames = await loadFramesFromWav(slot.value);
    }
  });
  await Promise.all(promises);
};

const updateSlotPartials = async (slotIndex) => {
  const slot = selectedTables[slotIndex];
  slot.frames = null;
  await ensureSelectedPartials();
  buildVoicePool();
};

const updateSlotModulation = (slotIndex) => {
  const slot = selectedTables[slotIndex];
  slot.lfoRate = toNumber(slot.lfoRate, 0);
  slot.lfoAmount = toNumber(slot.lfoAmount, 0);
  voicePool.forEach((voice) => {
    const entry = voice.slots?.[slotIndex];
    if (!entry) return;
    const frameCount = Math.max(slot.frames?.length || 0, 1);
    entry.lfo.frequency.value = slot.lfoRate;
    entry.lfo.min = 0;
    entry.lfo.max = frameCount > 1 ? (frameCount - 1) * slot.lfoAmount : 0;
  });
};

const getVectorWeights = () => {
  const x = Math.min(Math.max(toNumber(vectorPos.x, 0.5), 0), 1);
  const y = Math.min(Math.max(toNumber(vectorPos.y, 0.5), 0), 1);
  return [
    (1 - x) * (1 - y),
    x * (1 - y),
    (1 - x) * y,
    x * y
  ];
};

const activeVoices = new Map();
const laneVoices = Array.from({ length: rows }, () => null);
let voicePool = [];
let vectorLfoFrame;
let vectorLfoLast = 0;
const vectorLfoPhase = reactive({ x: 0, y: 0 });

const disposePool = () => {
  voicePool.forEach((voice) => disposeVoice(voice));
  voicePool = [];
};

const buildVoicePool = () => {
  disposePool();
  const hasAny = selectedTables.some((slot) => slot.frames?.length);
  if (!hasAny) return;

  for (let i = 0; i < POLY_VOICES; i += 1) {
    const envelope = new Tone.AmplitudeEnvelope({
      attack: toNumber(ampEnv.attack, 0.01),
      decay: toNumber(ampEnv.decay, 0.08),
      sustain: toNumber(ampEnv.sustain, 0.82),
      release: toNumber(ampEnv.release, 0.32)
    });

    const mix = new Tone.Gain();
    let filterNode;
    if (filterType.value === 'comb') {
      filterNode = new Tone.FeedbackCombFilter({
        delayTime: toNumber(combParams.delayTime, 0.02),
        resonance: toNumber(combParams.resonance, 0.6),
        dampening: toNumber(combParams.dampening, 3000)
      });
    } else {
      filterNode = new Tone.Filter({
        type: 'lowpass',
        frequency: toNumber(moogParams.cutoff, 1200),
        Q: toNumber(moogParams.resonance, 0.5) * 12,
        rolloff: -24
      });
      const drive = toNumber(moogParams.drive, 0.5);
      mix.gain.value = Math.max(0.2, 1 + drive * 0.8);
    }

    mix.connect(filterNode);
    filterNode.connect(envelope);
    envelope.connect(masterGain);
    const slots = selectedTables.map((slot) => {
      if (!slot.frames?.length) return null;
      const gain = new Tone.Gain(0).connect(mix);
      const frames = slot.frames;
      const frameCount = Math.max(frames.length, 1);
      const lfo = new Tone.LFO({
        frequency: toNumber(slot.lfoRate, 0.15),
        min: 0,
        max: frameCount > 1 ? (frameCount - 1) * toNumber(slot.lfoAmount, 1) : 0
      }).start();
      const osc = new Tone.Oscillator({
        type: 'custom',
        partials: frames[0] || [1]
      }).connect(gain);
      const partialLoop = frameCount > 1
        ? new Tone.Loop(() => {
            const pos = lfo.value;
            const idx = Math.floor(pos);
            const nextIdx = Math.min(idx + 1, frameCount - 1);
            const t = Math.min(Math.max(pos - idx, 0), 1);
            osc.partials = morphPartials(frames[idx], frames[nextIdx], t);
          }, '32n').start(0)
        : null;
      osc.start();
      return { osc, gain, lfo, partialLoop };
    });

    voicePool.push({
      envelope,
      filter: filterNode,
      mix,
      slots,
      busy: false,
      note: null
    });
  }
};

const updateActiveVoiceGains = () => {
  const weights = getVectorWeights();
  const now = Tone.now();
  voicePool.forEach((voice) => {
    if (!voice.busy) return;
    voice.slots?.forEach((entry, idx) => {
      if (entry?.gain) {
        const w = toNumber(weights[idx], 0);
        if (Number.isFinite(w)) {
          entry.gain.gain.setTargetAtTime(w, now, 0.03);
        }
      }
    });
  });
};

const triggerNoteOn = async (note, velocity = 0.9, time = Tone.now()) => {
  await ensureAudio();
  await ensureSelectedPartials();
  if (!voicePool.length) buildVoicePool();
  if (!voicePool.length) return null;

  const weights = getVectorWeights();
  const freq = Tone.Frequency(note).toFrequency();
  const now = time ?? Tone.now();

  const voice = voicePool.find((v) => !v.busy) || voicePool[0];
  voice.busy = true;
  voice.note = note;

  // filter envelope prep
  if (voice.filter) {
    if (filterType.value === 'comb') {
      const base = toNumber(combParams.dampening, 3000);
      const peak = Math.max(10, base + toNumber(filterEnv.amount, 0));
      const attackEnd = now + toNumber(filterEnv.attack, 0.01);
      const decayEnd = attackEnd + toNumber(filterEnv.decay, 0.1);
      const damp = voice.filter.dampening;
      if (damp?.cancelAndHoldAtTime && damp?.linearRampToValueAtTime) {
        damp.cancelAndHoldAtTime(now);
        damp.setValueAtTime(base, now);
        damp.linearRampToValueAtTime(peak, attackEnd);
        damp.linearRampToValueAtTime(base, decayEnd);
      } else if (damp?.setValueAtTime) {
        damp.setValueAtTime(base, now);
        damp.setValueAtTime(peak, attackEnd);
        damp.setValueAtTime(base, decayEnd);
      } else {
        voice.filter.dampening = base;
      }
    } else {
      const base = toNumber(moogParams.cutoff, 1200);
      const peak = Math.max(50, base + toNumber(filterEnv.amount, 0));
      const sustainLevel = base + toNumber(filterEnv.amount, 0) * toNumber(filterEnv.sustain, 0.5);
      const attackEnd = now + toNumber(filterEnv.attack, 0.01);
      const decayEnd = attackEnd + toNumber(filterEnv.decay, 0.1);
      const freq = voice.filter.frequency;
      if (freq?.cancelAndHoldAtTime) {
        freq.cancelAndHoldAtTime(now);
      }
      if (freq?.setValueAtTime && freq?.linearRampToValueAtTime) {
        freq.setValueAtTime(base, now);
        freq.linearRampToValueAtTime(peak, attackEnd);
        freq.linearRampToValueAtTime(sustainLevel, decayEnd);
      } else if (freq?.setValueAtTime) {
        freq.setValueAtTime(base, now);
        freq.setValueAtTime(peak, attackEnd);
        freq.setValueAtTime(sustainLevel, decayEnd);
      } else {
        voice.filter.frequency.value = sustainLevel;
      }
    }
  }

  voice.slots?.forEach((entry, idx) => {
    if (!entry) return;
    const w = toNumber(weights[idx], 0);
    if (Number.isFinite(w)) {
      entry.gain.gain.setTargetAtTime(w, now, 0.03);
    }
    entry.osc.frequency.setValueAtTime(freq, now);
  });

  voice.envelope.triggerAttack(now, velocity);

  const voices = activeVoices.get(note) || [];
  voices.push(voice);
  activeVoices.set(note, voices);
  return voice;
};

const triggerNoteOff = (note, time = Tone.now(), voiceRef = null) => {
  const voices = activeVoices.get(note);
  if (!voices || voices.length === 0) return;
  let voice;
  if (voiceRef) {
    const idx = voices.indexOf(voiceRef);
    voice = idx >= 0 ? voices.splice(idx, 1)[0] : voices.shift();
  } else {
    voice = voices.shift();
  }
  if (voice?.envelope) {
    voice.envelope.triggerRelease(time);
    const clearDelay = Math.max((time + 0.35 - Tone.now()) * 1000, 0);
    setTimeout(() => {
      voice.busy = false;
      voice.note = null;
    }, clearDelay);
  }
  if (!voices.length) {
    activeVoices.delete(note);
  }
};

const handleSequencerStep = async (time) => {
  const stepDuration = Tone.Time('16n').toSeconds();
  const noteDuration = Math.max(stepDuration * 0.8, 0.04);
  const stepsThisLoop = activeSteps.value;
  if (currentStep.value >= stepsThisLoop) currentStep.value = 0;
  const stepIndex = currentStep.value;
  for (let r = 0; r < rows; r += 1) {
    const prior = laneVoices[r];
    if (prior) {
      triggerNoteOff(prior.note, time, prior.voice);
      laneVoices[r] = null;
    }
    const cell = stepGrid[r][stepIndex];
    if (cell.active) {
      const voice = await triggerNoteOn(cell.note, 0.82, time);
      if (voice) {
        laneVoices[r] = { note: cell.note, voice };
        triggerNoteOff(cell.note, time + noteDuration, voice);
      }
    }
  }
  currentStep.value = (stepIndex + 1) % stepsThisLoop;
};

const syncSequencerPage = () => {
  if (!sequencer) return;
  const start = page.value * stepsPerPage;
  const end = start + stepsPerPage;
  const data = stepGrid.map((row) => row.slice(start, end).map((cell) => (cell.active ? 1 : 0)));
  sequencer.matrix.set.all(data);
};

const startSequencer = async () => {
  await ensureAudio();
  await ensureSelectedPartials();
  if (!voicePool.length) buildVoicePool();
  Tone.Transport.bpm.value = bpm.value;
  if (!loop) {
    loop = new Tone.Loop(handleSequencerStep, '16n');
  }
  loop.start(0);
  Tone.Transport.start();
  isSequencerRunning.value = true;
};

const stopSequencer = () => {
  if (loop) loop.stop();
  Tone.Transport.stop();
  // hard stop all voices when transport stops
  const now = Tone.now();
  activeVoices.forEach((voices, note) => {
    [...voices].forEach((voice) => triggerNoteOff(note, now, voice));
  });
  laneVoices.fill(null);
  isSequencerRunning.value = false;
  currentStep.value = 0;
};

const handleSequencerChange = ({ row, column, state }) => {
  const globalColumn = page.value * stepsPerPage + column;
  stepGrid[row][globalColumn].active = !!state;
  selectedCell.row = row;
  selectedCell.column = globalColumn;
  selectedNote.value = stepGrid[row][globalColumn].note;
};

const applySelectedNote = () => {
  stepGrid[selectedCell.row][selectedCell.column].note = selectedNote.value;
};

const handlePianoInput = async ({ note, state }) => {
  const name = Tone.Frequency(note, 'midi').toNote();
  if (state) {
    await triggerNoteOn(name, 0.9);
  } else {
    triggerNoteOff(name);
  }
};

const applyFilterSettings = () => {
  voicePool.forEach((voice) => {
    if (!voice.filter) return;
    if (filterType.value === 'comb') {
      voice.filter.delayTime.value = toNumber(combParams.delayTime, 0.02);
      voice.filter.resonance.value = toNumber(combParams.resonance, 0.6);
      const dampVal = toNumber(combParams.dampening, 3000);
      if (voice.filter.dampening?.setValueAtTime) {
        voice.filter.dampening.setValueAtTime(dampVal, Tone.now());
      } else {
        voice.filter.dampening = dampVal;
      }
    } else {
      voice.filter.frequency.value = toNumber(moogParams.cutoff, 1200);
      voice.filter.Q.value = toNumber(moogParams.resonance, 0.5) * 12;
    }
  });
};

onMounted(() => {
  if (gainSliderHost.value) {
    const slider = new Nexus.Slider(gainSliderHost.value, {
      size: [32, 180],
      min: -24,
      max: 6,
      step: 0.5,
      value: gainValue.value
    });

    slider.on('change', ({ value }) => {
      gainValue.value = value;
      Tone.Destination.volume.rampTo(value, 0.1);
    });
  }

  if (xyHost.value) {
    joystick = new Nexus.Position(xyHost.value, {
      size: [240, 240]
    });
    joystick.on('change', ({ x, y }) => {
      const invY = 1 - y;
      vectorBase.x = x;
      vectorBase.y = invY;
      vectorPos.x = x;
      vectorPos.y = invY; // invert Y to mimic joystick feel
    });
  }

  if (pianoHost.value) {
    piano = new Nexus.Piano(pianoHost.value, {
      size: [720, 120],
      lowNote: 48,
      highNote: 72
    });
    piano.on('change', handlePianoInput);
  }

  if (sequencerHost.value) {
    sequencer = new Nexus.Sequencer(sequencerHost.value, {
      size: [720, 240],
      rows,
      columns: stepsPerPage
    });
    sequencer.on('change', handleSequencerChange);
    syncSequencerPage();
  }

  startVectorLfos();
});

onBeforeUnmount(() => {
  stopSequencer();
  [joystick, piano, sequencer].forEach((widget) => {
    if (widget && typeof widget.destroy === 'function') widget.destroy();
  });
  if (vectorLfoFrame) cancelAnimationFrame(vectorLfoFrame);
  vectorLfoFrame = null;
});

watch(page, () => syncSequencerPage());
watch(bpm, (next) => {
  if (isSequencerRunning.value) {
    Tone.Transport.bpm.rampTo(next, 0.2);
  }
});
watch(
  () => [vectorPos.x, vectorPos.y],
  () => updateActiveVoiceGains()
);
watch(
  () => [vectorLfo.xRate, vectorLfo.xAmount, vectorLfo.yRate, vectorLfo.yAmount],
  () => updateVectorLfos()
);
watch(filterType, () => {
  buildVoicePool();
});
watch(
  () => [moogParams.cutoff, moogParams.resonance, moogParams.drive, combParams.delayTime, combParams.resonance, combParams.dampening],
  () => applyFilterSettings()
);
watch(
  () => [ampEnv.attack, ampEnv.decay, ampEnv.sustain, ampEnv.release, filterEnv.attack, filterEnv.decay, filterEnv.sustain, filterEnv.release, filterEnv.amount],
  () => buildVoicePool()
);
</script>

<template>
  <v-app>
    <v-main class="bg-grey-lighten-4">
      <v-container class="py-8">
        <v-row>
          <v-col cols="12" lg="8">
            <v-card>
              <v-card-title class="d-flex align-center justify-space-between">
                <div>
                  <div class="text-h6">Vector Wavetable Synth</div>
                  <div class="text-subtitle-2 text-medium-emphasis">
                    Four-slot vector oscillator blended by the joystick.
                  </div>
                </div>
                <div class="d-flex align-center ga-4">
                  <div class="text-caption text-medium-emphasis text-right">
                    Output gain<br />
                    {{ gainValue }} dB
                  </div>
                  <div class="nexus-vert" ref="gainSliderHost" />
                </div>
              </v-card-title>

              <v-card-text>
                <v-row class="mb-4">
                  <v-col cols="12" md="6">
                    <div class="text-subtitle-1 font-weight-medium mb-2">Wavetables</div>
                    <v-row dense>
                      <v-col v-for="(slot, idx) in selectedTables" :key="slot.label" cols="12" sm="6">
                        <v-select
                          :label="slot.label"
                          :items="wavetableOptions"
                          item-title="label"
                          item-value="value"
                          v-model="slot.value"
                          density="comfortable"
                          variant="outlined"
                          hide-details
                          @update:model-value="() => updateSlotPartials(idx)"
                        />
                        <v-text-field
                          class="mt-2"
                          label="LFO rate (Hz)"
                          type="number"
                          variant="outlined"
                          density="comfortable"
                          hide-details
                          :min="0"
                          :step="0.01"
                          v-model.number="slot.lfoRate"
                          @update:model-value="() => updateSlotModulation(idx)"
                        />
                        <v-slider
                          class="mt-2"
                          label="LFO amount"
                          v-model.number="slot.lfoAmount"
                          min="0"
                          max="1"
                          step="0.01"
                          density="comfortable"
                          hide-details
                          thumb-label
                          color="primary"
                          @update:model-value="() => updateSlotModulation(idx)"
                        />
                      </v-col>
                    </v-row>
                  </v-col>

                  <v-col cols="12" md="6">
                    <div class="text-subtitle-1 font-weight-medium mb-2">
                      Vector Joystick (Nexus Position)
                    </div>
                    <div class="nexus-pad" ref="xyHost" />
                    <div class="text-body-2 text-medium-emphasis mt-2">
                      X/Y mix → A/B/C/D: {{ vectorPos.x.toFixed(2) }} / {{ vectorPos.y.toFixed(2) }}
                    </div>
                    <v-divider class="my-2" />
                    <div class="text-subtitle-2 font-weight-medium mb-1">
                      Vector LFOs
                    </div>
                    <v-row dense>
                      <v-col cols="6">
                        <v-text-field
                          label="X rate (Hz)"
                          type="number"
                          v-model.number="vectorLfo.xRate"
                          variant="outlined"
                          density="comfortable"
                          hide-details
                          :step="0.01"
                          :min="0"
                        />
                      </v-col>
                      <v-col cols="6">
                        <v-slider
                          label="X amount"
                          v-model.number="vectorLfo.xAmount"
                          min="0"
                          max="1"
                          step="0.01"
                          density="comfortable"
                          hide-details
                          thumb-label
                          color="primary"
                        />
                      </v-col>
                      <v-col cols="6">
                        <v-text-field
                          label="Y rate (Hz)"
                          type="number"
                          v-model.number="vectorLfo.yRate"
                          variant="outlined"
                          density="comfortable"
                          hide-details
                          :step="0.01"
                          :min="0"
                        />
                      </v-col>
                      <v-col cols="6">
                        <v-slider
                          label="Y amount"
                          v-model.number="vectorLfo.yAmount"
                          min="0"
                          max="1"
                          step="0.01"
                          density="comfortable"
                          hide-details
                          thumb-label
                          color="primary"
                        />
                      </v-col>
                    </v-row>
                  </v-col>
                </v-row>

                <v-divider class="my-4" />

                <div class="text-subtitle-1 font-weight-medium mb-2">Keyboard (Nexus Piano)</div>
                <div class="nexus-wide" ref="pianoHost" />
                <v-alert
                  class="mt-4"
                  density="comfortable"
                  variant="tonal"
                  :type="isAudioReady ? 'success' : 'info'"
                >
                  {{ isAudioReady ? 'Audio unlocked. Play the keys or sequencer.' : 'Tap any control to unlock audio.' }}
                </v-alert>
              </v-card-text>
            </v-card>
          </v-col>

          <v-col cols="12" lg="4">
            <v-card class="h-100">
              <v-card-title class="text-h6">Step Sequencer</v-card-title>
              <v-card-subtitle>16 steps × 8 pages • 10 lanes</v-card-subtitle>
              <v-card-text>
                <v-row dense>
                  <v-col cols="6">
                    <v-text-field
                      label="BPM"
                      type="number"
                      v-model.number="bpm"
                      min="60"
                      max="180"
                      variant="outlined"
                      density="comfortable"
                    />
                  </v-col>
                  <v-col cols="6" class="d-flex align-center justify-end ga-2">
                    <v-btn color="primary" @click="startSequencer" :disabled="isSequencerRunning">
                      Start
                    </v-btn>
                    <v-btn color="secondary" variant="tonal" @click="stopSequencer" :disabled="!isSequencerRunning">
                      Stop
                    </v-btn>
                  </v-col>
                </v-row>

                <v-row dense class="mt-2 mb-3">
                  <v-col cols="6">
                    <v-select
                      label="Page"
                      :items="Array.from({ length: totalPages }, (_, i) => ({ title: `Page ${i + 1}`, value: i }))"
                      v-model="page"
                      variant="outlined"
                      density="comfortable"
                      hide-details
                    />
                  </v-col>
                  <v-col cols="6" class="d-flex align-center justify-end text-medium-emphasis">
                    Step: {{ currentStep + 1 }} / {{ activeSteps }}
                  </v-col>
                </v-row>

                <div class="nexus-wide" ref="sequencerHost" />

                <v-divider class="my-4" />

                <div class="text-subtitle-2 font-weight-medium mb-2">
                  Step Note Assignment
                </div>
                <div class="text-body-2 text-medium-emphasis mb-2">
                  Selected: Row {{ selectedCell.row + 1 }}, Step {{ selectedCell.column + 1 }}
                </div>
                <v-select
                  :items="NOTE_POOL"
                  v-model="selectedNote"
                  label="Note"
                  variant="outlined"
                  density="comfortable"
                  hide-details
                />
                <v-btn class="mt-3" block color="primary" @click="applySelectedNote">
                  Set note for selected step
                </v-btn>
              </v-card-text>
            </v-card>
          </v-col>

          <v-col cols="12">
            <v-card>
              <v-card-title class="text-h6">Filters & Envelopes</v-card-title>
              <v-card-text>
                <v-row>
                  <v-col cols="12" md="4">
                    <v-select
                      label="Filter type"
                      :items="[{ title: 'Moog (LPF)', value: 'moog' }, { title: 'Comb', value: 'comb' }]"
                      v-model="filterType"
                      variant="outlined"
                      density="comfortable"
                      hide-details
                    />

                    <template v-if="filterType === 'moog'">
                      <v-text-field
                        class="mt-3"
                        label="Cutoff (Hz)"
                        type="number"
                        v-model.number="moogParams.cutoff"
                        variant="outlined"
                        density="comfortable"
                        hide-details
                      />
                      <v-slider
                        class="mt-3"
                        label="Resonance"
                        v-model.number="moogParams.resonance"
                        min="0"
                        max="1"
                        step="0.01"
                        density="comfortable"
                        hide-details
                        thumb-label
                        color="primary"
                      />
                      <v-slider
                        class="mt-3"
                        label="Drive"
                        v-model.number="moogParams.drive"
                        min="0"
                        max="1"
                        step="0.01"
                        density="comfortable"
                        hide-details
                        thumb-label
                        color="primary"
                      />
                    </template>

                    <template v-else>
                      <v-text-field
                        class="mt-3"
                        label="Delay (s)"
                        type="number"
                        v-model.number="combParams.delayTime"
                        variant="outlined"
                        density="comfortable"
                        hide-details
                        :step="0.001"
                        :min="0.001"
                      />
                      <v-slider
                        class="mt-3"
                        label="Resonance"
                        v-model.number="combParams.resonance"
                        min="0"
                        max="0.99"
                        step="0.01"
                        density="comfortable"
                        hide-details
                        thumb-label
                        color="primary"
                      />
                      <v-text-field
                        class="mt-3"
                        label="Dampening (Hz)"
                        type="number"
                        v-model.number="combParams.dampening"
                        variant="outlined"
                        density="comfortable"
                        hide-details
                      />
                    </template>
                  </v-col>

                  <v-col cols="12" md="4">
                    <div class="text-subtitle-2 font-weight-medium mb-2">Amp Envelope</div>
                    <v-row dense>
                      <v-col cols="6">
                        <v-text-field
                          label="Attack"
                          type="number"
                          v-model.number="ampEnv.attack"
                          variant="outlined"
                          density="comfortable"
                          hide-details
                          :step="0.005"
                        />
                      </v-col>
                      <v-col cols="6">
                        <v-text-field
                          label="Decay"
                          type="number"
                          v-model.number="ampEnv.decay"
                          variant="outlined"
                          density="comfortable"
                          hide-details
                          :step="0.01"
                        />
                      </v-col>
                      <v-col cols="6">
                        <v-slider
                          label="Sustain"
                          v-model.number="ampEnv.sustain"
                          min="0"
                          max="1"
                          step="0.01"
                          density="comfortable"
                          hide-details
                          thumb-label
                          color="primary"
                        />
                      </v-col>
                      <v-col cols="6">
                        <v-text-field
                          label="Release"
                          type="number"
                          v-model.number="ampEnv.release"
                          variant="outlined"
                          density="comfortable"
                          hide-details
                          :step="0.01"
                        />
                      </v-col>
                    </v-row>
                  </v-col>

                  <v-col cols="12" md="4">
                    <div class="text-subtitle-2 font-weight-medium mb-2">Filter Envelope</div>
                    <v-row dense>
                      <v-col cols="6">
                        <v-text-field
                          label="Attack"
                          type="number"
                          v-model.number="filterEnv.attack"
                          variant="outlined"
                          density="comfortable"
                          hide-details
                          :step="0.005"
                        />
                      </v-col>
                      <v-col cols="6">
                        <v-text-field
                          label="Decay"
                          type="number"
                          v-model.number="filterEnv.decay"
                          variant="outlined"
                          density="comfortable"
                          hide-details
                          :step="0.01"
                        />
                      </v-col>
                      <v-col cols="6">
                        <v-slider
                          label="Sustain"
                          v-model.number="filterEnv.sustain"
                          min="0"
                          max="1"
                          step="0.01"
                          density="comfortable"
                          hide-details
                          thumb-label
                          color="primary"
                        />
                      </v-col>
                      <v-col cols="6">
                        <v-text-field
                          label="Release"
                          type="number"
                          v-model.number="filterEnv.release"
                          variant="outlined"
                          density="comfortable"
                          hide-details
                          :step="0.01"
                        />
                      </v-col>
                      <v-col cols="12">
                        <v-text-field
                          label="Amount"
                          type="number"
                          v-model.number="filterEnv.amount"
                          variant="outlined"
                          density="comfortable"
                          hide-details
                        />
                      </v-col>
                    </v-row>
                  </v-col>
                </v-row>
              </v-card-text>
            </v-card>
          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>

<style scoped>
.nexus-vert {
  width: 48px;
  height: 200px;
}

.nexus-pad {
  width: 260px;
  height: 260px;
}

.nexus-wide {
  width: 100%;
}

.ga-2 {
  gap: 8px;
}

.ga-4 {
  gap: 16px;
}
</style>
