const TOTAL_STEPS = 16;
const NOTE_NAMES = ["C4", "D4", "E4", "F4", "G4", "A4", "B4", "C5", "D5", "E5", "F5", "G5"];

const instrumentNames = [
  "Acoustic Piano",
  "Electric Piano",
  "Warm Piano",
  "Bright Piano",
  "Grand Piano",
  "Upright Piano",
  "Celesta",
  "Bell Piano",
  "Harpsichord",
  "Rhodes",
  "Clavichord",
  "Organ",
  "Church Organ",
  "Jazz Organ",
  "Synth Organ",
  "String Guitar",
  "Electric Guitar",
  "Clean Guitar",
  "Distorted Guitar",
  "Acoustic Guitar",
  "Bass Guitar",
  "Fretless Bass",
  "Upright Bass",
  "Synth Bass",
  "Sub Bass",
  "Plucked Bass",
  "Violin",
  "Viola",
  "Cello",
  "Double Bass",
  "Trumpet",
  "Trombone",
  "Saxophone",
  "Flute",
  "Clarinet",
  "Oboe",
  "Bassoon",
  "Pan Flute",
  "Recorder",
  "Piccolo",
  "Drum Kit",
  "Snare Drum",
  "Kick Drum",
  "Tom Drum",
  "Hi-Hat",
  "Cymbal Crash",
  "Rim Shot",
  "Conga",
  "Bongo",
  "Marimba",
  "Xylophone",
  "Vibraphone",
  "Glockenspiel",
  "Kalimba",
  "Handpan",
  "Steel Drum",
  "Shamisen",
  "Sitar",
  "Banjo",
  "Mandolin",
  "Ukulele",
  "Harp",
  "Synth Lead",
  "Synth Pad",
  "Analog Lead",
  "Poly Synth",
  "FM Synth",
  "Pulse Synth",
  "Glue Synth",
  "Chime Synth",
  "Noise Synth",
  "Drone Synth",
  "Brass Pad",
  "Choir Pad",
  "String Pad",
  "Dark Pad",
  "Cosmic Pad",
  "Soft Bell",
  "Glass Bell",
  "Crystal Chime",
  "Metal Chime",
  "Water Drop",
  "Birdcall",
  "Thunder",
  "Rain Cymbal",
  "Space Echo",
  "Pulse Drum",
  "Deep Bass",
  "Studio Kit",
];

if (instrumentNames.length !== 89) {
  throw new Error(`Expected 89 instruments, found ${instrumentNames.length}.`);
}

const instrumentLibrary = instrumentNames.map((name, index) => {
  const waveForms = ["sine", "triangle", "square", "sawtooth"];
  const group = /Drum|Kit|Snare|Kick|Tom|Hi-Hat|Cymbal|Rim|Conga|Bongo|Percussion|Thunder|Rain|Pulse/i.test(name)
    ? "percussion"
    : "tone";

  return {
    id: index,
    name,
    group,
    wave: waveForms[index % waveForms.length],
    detune: (index % 15) * 3 - 18,
    cutoff: 500 + (index % 22) * 120,
    q: 0.8 + (index % 7) * 0.6,
    attack: 0.01 + (index % 9) * 0.012,
    release: 0.18 + (index % 10) * 0.04,
    gain: 0.18 + (index % 8) * 0.05,
    bend: index % 2 === 0 ? 1 : 1.3,
    accent: 0.4 + (index % 12) * 0.05,
  };
});

const ui = {
  grid: document.querySelector("#instrumentGrid"),
  trackList: document.querySelector("#trackList"),
  playButton: document.querySelector("#playButton"),
  stopButton: document.querySelector("#stopButton"),
  addTrackButton: document.querySelector("#addTrackButton"),
  previewSelected: document.querySelector("#previewSelected"),
  tempoSlider: document.querySelector("#tempoSlider"),
  tempoValue: document.querySelector("#tempoValue"),
};

const state = {
  tempo: Number(ui.tempoSlider.value),
  selectedInstrument: 0,
  tracks: [],
  currentStep: 0,
  phaseTimer: null,
  isPlaying: false,
};

const audioContext = new (window.AudioContext || window.webkitAudioContext)();
const masterGain = audioContext.createGain();
masterGain.gain.value = 0.18;
masterGain.connect(audioContext.destination);

function noteToFrequency(noteName) {
  const match = noteName.match(/^([A-G])(#?)(\d)$/);
  if (!match) return 220;

  const [, letter, accidental, octave] = match;
  const noteMap = { C: 0, "C#": 1, D: 2, "D#": 3, E: 4, F: 5, "F#": 6, G: 7, "G#": 8, A: 9, "A#": 10, B: 11 };
  const midi = noteMap[`${letter}${accidental}`] + (Number(octave) + 1) * 12 + 12;
  return 440 * Math.pow(2, (midi - 69) / 12);
}

function buildTrack(instrumentId = 0) {
  return {
    id: crypto.randomUUID(),
    instrumentId,
    volume: 0.7,
    rootNoteIndex: 0,
    steps: Array(TOTAL_STEPS).fill(false),
  };
}

function renderInstrumentGrid() {
  ui.grid.innerHTML = "";

  instrumentLibrary.forEach((instrument) => {
    const button = document.createElement("button");
    button.type = "button";
    button.className = `instrument-chip ${state.selectedInstrument === instrument.id ? "selected" : ""}`;
    button.innerHTML = `<span>${instrument.name}</span><small>#${instrument.id + 1}</small>`;
    button.addEventListener("click", () => {
      state.selectedInstrument = instrument.id;
      if (state.tracks.length > 0) {
        const selectedTrack = state.tracks[state.tracks.length - 1];
        selectedTrack.instrumentId = instrument.id;
      }
      renderInstrumentGrid();
      renderTracks();
    });
    ui.grid.appendChild(button);
  });
}

function renderTracks() {
  ui.trackList.innerHTML = "";

  state.tracks.forEach((track, trackIndex) => {
    const instrument = instrumentLibrary[track.instrumentId];
    const row = document.createElement("div");
    row.className = "track";
    row.innerHTML = `
      <div class="track-header">
        <span class="track-label">Track ${trackIndex + 1}</span>
        <select aria-label="Instrument for track ${trackIndex + 1}">
          ${instrumentLibrary
            .map(
              (item, index) =>
                `<option value="${index}" ${index === track.instrumentId ? "selected" : ""}>${item.name}</option>`
            )
            .join("")}
        </select>
        <input type="range" min="0.1" max="1" step="0.05" value="${track.volume}" aria-label="Track volume" />
        <button class="remove-track" type="button" aria-label="Remove track">×</button>
      </div>
    `;

    const stepGrid = document.createElement("div");
    stepGrid.className = "step-grid";

    for (let step = 0; step < TOTAL_STEPS; step += 1) {
      const button = document.createElement("button");
      button.type = "button";
      button.className = `step ${track.steps[step] ? "active" : ""} ${step === state.currentStep && state.isPlaying ? "current" : ""}`;
      button.textContent = step % 4 === 0 ? step + 1 : "";
      button.title = `Step ${step + 1}`;
      button.addEventListener("click", () => {
        track.steps[step] = !track.steps[step];
        renderTracks();
      });
      stepGrid.appendChild(button);
    }

    const select = row.querySelector("select");
    const volumeInput = row.querySelector("input[type='range']");
    const removeButton = row.querySelector(".remove-track");

    select.addEventListener("change", (event) => {
      track.instrumentId = Number(event.target.value);
      renderTracks();
    });

    volumeInput.addEventListener("input", (event) => {
      track.volume = Number(event.target.value);
    });

    removeButton.addEventListener("click", () => {
      state.tracks.splice(trackIndex, 1);
      renderTracks();
    });

    row.appendChild(stepGrid);
    ui.trackList.appendChild(row);
  });
}

function playTone(frequency, instrument, duration = 0.22) {
  const now = audioContext.currentTime;

  const oscillator = audioContext.createOscillator();
  const gainNode = audioContext.createGain();
  const filterNode = audioContext.createBiquadFilter();

  oscillator.type = instrument.wave;
  oscillator.frequency.setValueAtTime(frequency * instrument.bend, now);
  oscillator.detune.value = instrument.detune;

  filterNode.type = "lowpass";
  filterNode.frequency.value = instrument.cutoff;
  filterNode.Q.value = instrument.q;

  gainNode.gain.setValueAtTime(0.0001, now);
  gainNode.gain.exponentialRampToValueAtTime(instrument.gain, now + instrument.attack);
  gainNode.gain.exponentialRampToValueAtTime(0.0001, now + duration + instrument.release);

  oscillator.connect(filterNode);
  filterNode.connect(gainNode);
  gainNode.connect(masterGain);

  oscillator.start(now);
  oscillator.stop(now + duration + instrument.release + 0.04);
}

function playDrum(instrument, frequency, duration = 0.16) {
  const now = audioContext.currentTime;
  const noiseBuffer = audioContext.createBuffer(1, audioContext.sampleRate * duration, audioContext.sampleRate);
  const channelData = noiseBuffer.getChannelData(0);

  for (let i = 0; i < channelData.length; i += 1) {
    channelData[i] = Math.random() * 2 - 1;
  }

  const source = audioContext.createBufferSource();
  const filter = audioContext.createBiquadFilter();
  const gain = audioContext.createGain();

  source.buffer = noiseBuffer;
  filter.type = "highpass";
  filter.frequency.value = 220 + instrument.cutoff * 0.6;
  gain.gain.setValueAtTime(0.0001, now);
  gain.gain.exponentialRampToValueAtTime(instrument.gain * 0.95, now + 0.005);
  gain.gain.exponentialRampToValueAtTime(0.0001, now + duration * 0.9);

  source.connect(filter);
  filter.connect(gain);
  gain.connect(masterGain);

  source.start(now);
  source.stop(now + duration * 1.1);

  if (frequency) {
    const tone = audioContext.createOscillator();
    const toneGain = audioContext.createGain();
    tone.type = "triangle";
    tone.frequency.setValueAtTime(frequency * 0.4, now);
    toneGain.gain.setValueAtTime(0.0001, now);
    toneGain.gain.exponentialRampToValueAtTime(instrument.gain * 0.5, now + 0.01);
    toneGain.gain.exponentialRampToValueAtTime(0.0001, now + duration);
    tone.connect(toneGain);
    toneGain.connect(masterGain);
    tone.start(now);
    tone.stop(now + duration);
  }
}

function triggerTrackStep(track) {
  const instrument = instrumentLibrary[track.instrumentId];
  const rootFrequency = noteToFrequency(NOTE_NAMES[track.rootNoteIndex]);

  if (instrument.group === "percussion") {
    playDrum(instrument, rootFrequency, 0.18);
    return;
  }

  playTone(rootFrequency, instrument, 0.22);
}

function playStep() {
  for (const track of state.tracks) {
    if (track.steps[state.currentStep]) {
      triggerTrackStep(track);
    }
  }

  state.currentStep = (state.currentStep + 1) % TOTAL_STEPS;
  renderTracks();
}

function togglePlayback() {
  if (state.isPlaying) {
    stopPlayback();
    return;
  }

  if (audioContext.state === "suspended") {
    audioContext.resume();
  }

  state.isPlaying = true;
  ui.playButton.textContent = "Pause";
  state.phaseTimer = window.setInterval(playStep, 60000 / state.tempo / 2);
  renderTracks();
}

function stopPlayback() {
  state.isPlaying = false;
  ui.playButton.textContent = "Play";
  if (state.phaseTimer) {
    window.clearInterval(state.phaseTimer);
    state.phaseTimer = null;
  }
  state.currentStep = 0;
  renderTracks();
}

function addTrack() {
  const nextTrack = buildTrack(state.selectedInstrument);
  state.tracks.push(nextTrack);
  renderTracks();
}

function previewSelectedInstrument() {
  const instrument = instrumentLibrary[state.selectedInstrument];
  const freq = noteToFrequency(NOTE_NAMES[0]);
  if (instrument.group === "percussion") {
    playDrum(instrument, freq, 0.2);
  } else {
    playTone(freq, instrument, 0.32);
  }
}

ui.playButton.addEventListener("click", togglePlayback);
ui.stopButton.addEventListener("click", stopPlayback);
ui.addTrackButton.addEventListener("click", addTrack);
ui.previewSelected.addEventListener("click", previewSelectedInstrument);

ui.tempoSlider.addEventListener("input", (event) => {
  state.tempo = Number(event.target.value);
  ui.tempoValue.textContent = `${state.tempo} BPM`;

  if (state.isPlaying) {
    stopPlayback();
    togglePlayback();
  }
});

for (let i = 0; i < 4; i += 1) {
  state.tracks.push(buildTrack(i % instrumentLibrary.length));
}

ui.tempoValue.textContent = `${state.tempo} BPM`;
renderInstrumentGrid();
renderTracks();

const trackSeed = [0, 15, 63, 41];
state.tracks = trackSeed.map((instrumentId) => {
  const track = buildTrack(instrumentId);
  track.steps[0] = true;
  track.steps[4] = true;
  track.steps[8] = true;
  track.steps[12] = true;
  return track;
});
renderTracks();

window.addEventListener("keydown", (event) => {
  if (event.code === "Space") {
    event.preventDefault();
    togglePlayback();
  }
});

renderInstrumentGrid();
renderTracks();
