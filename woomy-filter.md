layout: page
title: "Woomy filter"
permalink: /woomy-filter

<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Filtro de Voz Inkling - Splatoon</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background-color: #121212;
      color: #fff;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
      margin: 0;
      padding: 20px;
      box-sizing: border-box;
    }
    .card {
      background: #1e1e1e;
      padding: 24px 32px;
      border-radius: 12px;
      box-shadow: 0 4px 15px rgba(0,0,0,0.5);
      border: 2px solid #ff007f;
      width: 360px;
    }
    h1 {
      color: #ff007f;
      margin: 0 0 6px;
      text-align: center;
      font-size: 22px;
    }
    .subtitle {
      color: #aaa;
      font-size: 13px;
      text-align: center;
      margin-bottom: 20px;
    }
    fieldset {
      border: 1px solid #444;
      border-radius: 6px;
      margin: 0 0 16px;
      padding: 12px 14px 14px;
    }
    legend {
      color: #ccc;
      font-size: 13px;
      padding: 0 6px;
    }
    .control-row {
      display: grid;
      grid-template-columns: 80px 52px 1fr;
      align-items: center;
      gap: 10px;
      margin-bottom: 12px;
    }
    .control-row:last-child {
      margin-bottom: 0;
    }
    .control-row label {
      font-size: 13px;
      color: #bbb;
    }
    .control-row input[type="number"] {
      width: 100%;
      box-sizing: border-box;
      padding: 4px 6px;
      border-radius: 4px;
      border: 1px solid #555;
      background: #2a2a2a;
      color: #fff;
      font-size: 13px;
      text-align: right;
    }
    .control-row input[type="range"] {
      width: 100%;
      accent-color: #ff007f;
      cursor: pointer;
    }
    button {
      background-color: #ff007f;
      color: white;
      border: none;
      padding: 12px 24px;
      font-size: 16px;
      font-weight: bold;
      border-radius: 6px;
      cursor: pointer;
      transition: background 0.2s;
      width: 100%;
      margin-top: 4px;
    }
    button:hover {
      background-color: #e0006f;
    }
    #status {
      margin-top: 14px;
      font-weight: bold;
      color: #00ffcc;
      font-size: 13px;
      text-align: center;
      white-space: pre-line;
    }
    .hint {
      margin-top: 10px;
      font-size: 11px;
      color: #888;
      text-align: left;
      line-height: 1.4;
    }
  </style>
</head>
<body>

  <div class="card">
    <h1>Filtro Inkling 馃</h1>
    <p class="subtitle">Procesamiento de voz en tiempo real</p>

    <fieldset>
      <legend>Controles</legend>

      <div class="control-row">
        <label for="pitchNum">Pitch (st)</label>
        <input type="number" id="pitchNum" min="-12" max="12" step="0.5" value="0.0">
        <input type="range" id="pitchSlider" min="-12" max="12" step="0.5" value="0">
      </div>
    </fieldset>

    <fieldset>
      <legend>Salida</legend>

      <div class="control-row">
        <label for="volNum">Volumen (dB)</label>
        <input type="number" id="volNum" min="-24" max="12" step="0.1" value="0.0">
        <input type="range" id="volSlider" min="-24" max="12" step="0.1" value="0">
      </div>

      <div class="control-row">
        <label for="limNum">Limit. (dB)</label>
        <input type="number" id="limNum" min="-24" max="0" step="0.1" value="-6.0">
        <input type="range" id="limSlider" min="-24" max="0" step="0.1" value="-6">
      </div>
    </fieldset>

    <button id="toggleBtn">Activar Filtro</button>
    <div id="status">Estado: Desactivado</div>
    <div id="hint" class="hint"></div>
  </div>

  <script>
    let audioCtx = null;
    let micStream = null;
    let isRunning = false;

    let filter = null;
    let lfo = null;
    let lfoGain = null;
    let carrierOsc = null;
    let pitchNode = null;
    let setPitchRatio = null;
    let limiter = null;
    let masterGain = null;

    function createLimiter(ctx) {
      const comp = ctx.createDynamicsCompressor();
      comp.threshold.value = -6;
      comp.knee.value = 4;
      comp.ratio.value = 16;
      comp.attack.value = 0.003;
      comp.release.value = 0.22;
      return comp;
    }

    function createPitchShifter(ctx) {
      const len = 65536;
      const targetDelay = 12000;
      const node = ctx.createScriptProcessor(2048, 1, 1);
      const buffer = new Float32Array(len);

      let writePos = 0;
      let readPos = 0;
      let started = false;
      let pitch = 1;
      let targetPitch = 1;

      function sampleAt(pos) {
        const i0 = ((Math.floor(pos) % len) + len) % len;
        const i1 = (i0 + 1) % len;
        const frac = pos - Math.floor(pos);
        return buffer[i0] * (1 - frac) + buffer[i1] * frac;
      }

      node.onaudioprocess = (e) => {
        const input = e.inputBuffer.getChannelData(0);
        const output = e.outputBuffer.getChannelData(0);

        pitch += (targetPitch - pitch) * 0.04;

        if (Math.abs(pitch - 1) < 0.001) {
          output.set(input);
          return;
        }

        for (let i = 0; i < input.length; i++) {
          buffer[writePos % len] = input[i];
          writePos++;

          if (!started && writePos >= targetDelay) {
            readPos = writePos - targetDelay;
            started = true;
          }

          if (!started) {
            output[i] = 0;
            continue;
          }

          output[i] = sampleAt(readPos);

          const delay = writePos - readPos;
          const error = delay - targetDelay;
          readPos += pitch + error * 0.00006;
        }
      };

      return {
        node,
        setPitch(ratio) {
          targetPitch = clamp(ratio, 0.5, 2);
        }
      };
    }

    function describeMicError(err) {
      if (!navigator.mediaDevices) {
        return "Tu navegador no soporta acceso al micr贸fono.";
      }
      if (!window.isSecureContext) {
        return "Abre la p谩gina por http://localhost (no como archivo local).";
      }
      if (err.name === "NotAllowedError" || err.name === "PermissionDeniedError") {
        return "Permiso denegado. Haz clic en el candado de la barra de direcciones y permite el micr贸fono.";
      }
      if (err.name === "NotFoundError" || err.name === "DevicesNotFoundError") {
        return "No se encontr贸 ning煤n micr贸fono conectado.";
      }
      if (err.name === "NotReadableError" || err.name === "TrackStartError") {
        return "El micr贸fono est谩 en uso por otra aplicaci贸n.";
      }
      return err.message || "Error desconocido al iniciar el audio.";
    }

    async function stopAudio() {
      if (micStream) micStream.getTracks().forEach((track) => track.stop());
      if (audioCtx) await audioCtx.close();

      filter = lfo = lfoGain = carrierOsc = null;
      pitchNode = limiter = masterGain = null;
      setPitchRatio = null;
      audioCtx = micStream = null;
    }

    const toggleBtn = document.getElementById("toggleBtn");
    const statusDiv = document.getElementById("status");
    const hintDiv = document.getElementById("hint");

    if (location.protocol === "file:") {
      hintDiv.innerHTML =
        "鈿狅笍 Abriste el archivo directamente. Chrome/Edge bloquean el micr贸fono as铆.<br>" +
        "En PowerShell, desde el Escritorio ejecuta:<br>" +
        "<code>python -m http.server 8080</code><br>" +
        "Luego abre: <code>http://localhost:8080/inklingfilter.html</code>";
    } else if (!window.isSecureContext) {
      hintDiv.textContent = "鈿狅笍 El micr贸fono solo funciona en HTTPS o localhost.";
    }

    const controls = [
      { num: document.getElementById("pitchNum"), slider: document.getElementById("pitchSlider"), key: "pitch" },
      { num: document.getElementById("volNum"), slider: document.getElementById("volSlider"), key: "volume" },
      { num: document.getElementById("limNum"), slider: document.getElementById("limSlider"), key: "limiter" }
    ];

    function dbToGain(db) {
      return Math.pow(10, db / 20);
    }

    function clamp(value, min, max) {
      return Math.min(max, Math.max(min, value));
    }

    function getValues() {
      return {
        pitch: parseFloat(document.getElementById("pitchNum").value) || 0,
        volume: parseFloat(document.getElementById("volNum").value) || 0,
        limiter: parseFloat(document.getElementById("limNum").value) ?? -6
      };
    }

    function semitonesToRatio(st) {
      return Math.pow(2, st / 12);
    }

    function applyTone() {
      const { pitch, volume, limiter: limitDb } = getValues();

      if (masterGain) masterGain.gain.value = dbToGain(volume);
      if (limiter) limiter.threshold.value = limitDb;
      if (setPitchRatio) setPitchRatio(semitonesToRatio(pitch));
    }

    function syncPair(numEl, sliderEl, value) {
      const clamped = clamp(value, parseFloat(sliderEl.min), parseFloat(sliderEl.max));
      const fixed = clamped.toFixed(1);
      numEl.value = fixed;
      sliderEl.value = fixed;
    }

    function bindControl({ num, slider, key }) {
      const updateFrom = (source) => {
        const raw = parseFloat(source.value);
        if (Number.isNaN(raw)) return;

        syncPair(num, slider, raw);
        applyTone();
      };

      num.addEventListener("change", () => updateFrom(num));
      num.addEventListener("input", () => updateFrom(num));
      slider.addEventListener("input", () => updateFrom(slider));
    }

    controls.forEach(bindControl);

    toggleBtn.addEventListener("click", async () => {
      if (!isRunning) {
        try {
          if (!navigator.mediaDevices?.getUserMedia) {
            throw new Error("Navegador sin soporte para micr贸fono.");
          }
          if (!window.isSecureContext) {
            throw new Error("Se requiere abrir por http://localhost, no como archivo local.");
          }

          audioCtx = new (window.AudioContext || window.webkitAudioContext)();

          try {
            micStream = await navigator.mediaDevices.getUserMedia({
              audio: {
                echoCancellation: false,
                noiseSuppression: false,
                autoGainControl: false
              }
            });
          } catch (micErr) {
            throw Object.assign(micErr, { _isMicError: true });
          }

          await audioCtx.resume();
          const micInput = audioCtx.createMediaStreamSource(micStream);

          filter = audioCtx.createBiquadFilter();
          filter.type = "bandpass";
          filter.frequency.value = 1200;
          filter.Q.value = 5;

          lfo = audioCtx.createOscillator();
          lfoGain = audioCtx.createGain();
          lfo.type = "sine";
          lfo.frequency.value = 8;
          lfoGain.gain.value = 800;
          lfo.connect(lfoGain);
          lfoGain.connect(filter.frequency);
          lfo.start();

          // 3. Modulaci贸n en anillo 鈥?igual que la versi贸n original
          const ringModGain = audioCtx.createGain();
          const carrierDepth = audioCtx.createGain();
          carrierDepth.gain.value = 0.85;

          carrierOsc = audioCtx.createOscillator();
          carrierOsc.type = "sawtooth";
          carrierOsc.frequency.value = 35;
          carrierOsc.connect(carrierDepth);
          carrierDepth.connect(ringModGain.gain);
          carrierOsc.start();

          const pitchShifter = createPitchShifter(audioCtx);
          pitchNode = pitchShifter.node;
          setPitchRatio = pitchShifter.setPitch;

          limiter = createLimiter(audioCtx);

          masterGain = audioCtx.createGain();
          masterGain.gain.value = 1;

          // Cadena original + pitch / limitador / volumen al final
          micInput.connect(filter);
          filter.connect(ringModGain);
          ringModGain.connect(pitchNode);
          pitchNode.connect(limiter);
          limiter.connect(masterGain);
          masterGain.connect(audioCtx.destination);

          applyTone();

          isRunning = true;
          toggleBtn.innerText = "Detener Filtro";
          statusDiv.innerText = "Estado: Transmitiendo voz de Inkling 馃";
          statusDiv.style.color = "#ff007f";
        } catch (err) {
          console.error("Error al iniciar:", err);
          await stopAudio();
          isRunning = false;
          toggleBtn.innerText = "Activar Filtro";
          statusDiv.innerText = "Error: " + describeMicError(err);
          statusDiv.style.color = "#ff3333";
        }
      } else {
        await stopAudio();

        isRunning = false;
        toggleBtn.innerText = "Activar Filtro";
        statusDiv.innerText = "Estado: Desactivado";
        statusDiv.style.color = "#00ffcc";
      }
    });
  </script>
</body>
</html>
