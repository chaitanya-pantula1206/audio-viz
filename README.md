# SonicWave — WASM Audio Visualizer

A Chrome extension that captures audio playing in the active browser tab and draws a real time frequency visualizer for it. The frequency analysis (FFT) is written in C++ and compiled to WebAssembly, so the heavy math runs outside the JavaScript main thread instead of inside it.

## How it works

1. `chrome.tabCapture` grabs the audio stream from the active tab.
2. The stream is piped into a Web Audio `AnalyserNode`, which hands over raw time domain samples on every animation frame.
3. Those samples are passed into the WebAssembly module, which runs an iterative radix-2 Cooley-Tukey FFT (written in `core/src/fft.cpp`) and returns the frequency magnitudes.
4. The magnitudes are mapped onto a log frequency scale and drawn as a bar visualizer on a canvas.

If the WASM module fails to load for any reason, the extension falls back to the browser's built in `getFloatFrequencyData`, so the visualizer still works, just without the custom FFT engine. The status badge in the popup shows which engine is active.

## Tech stack

- **Frontend**: TypeScript, Vite, HTML Canvas
- **DSP engine**: C++17, compiled with Emscripten to WebAssembly
- **Extension**: Chrome Manifest V3, `tabCapture` and `activeTab` permissions

## Project structure

```
audio-viz/
├── core/                    # C++ FFT engine
│   ├── include/fft.h
│   └── src/
│       ├── fft.cpp          # radix-2 Cooley-Tukey FFT implementation
│       ├── wasm_interface.cpp   # C-linkage bridge exposed to JS via Emscripten
│       └── main.cpp         # native test harness (runs the FFT on a synthetic signal, no browser needed)
├── src/
│   ├── main.ts              # capture pipeline, canvas rendering, WASM bridging
│   └── background.ts        # extension background script
├── public/manifest.json     # Chrome extension manifest (compiled WASM output also lands here)
├── CMakeLists.txt           # builds fft_wasm (Emscripten) or fft_test (native), depending on toolchain
└── vite.config.ts
```

## Building it

You need Node.js and the [Emscripten SDK](https://emscripten.org/docs/getting_started/downloads.html) on your PATH for the WASM build.

**1. Compile the FFT engine to WebAssembly**

```bash
mkdir build && cd build
emcmake cmake ..
emmake make
```

This drops `fft_wasm.js` and `fft_wasm.wasm` into `public/`, where the frontend expects them.

**2. Build the extension**

```bash
npm install
npm run build
```

This outputs the popup, background script, and manifest into `dist/`.

**3. Load it in Chrome**

1. Go to `chrome://extensions`
2. Enable Developer mode
3. Click "Load unpacked" and select the `dist/` folder
4. Open a tab that's playing audio, click the extension icon, then "Start Capturing Tab"

If you only want to test the FFT implementation itself without touching the browser, the native build target works standalone:

```bash
mkdir build && cd build
cmake ..
make
./fft_test
```

This runs the FFT against a synthetic 440 Hz + 1000 Hz signal and prints the resulting frequency bins to the console, useful for checking the math without going through the extension at all.

## Limitations

- `tabCapture` only works on a tab that's currently playing audio and active; it won't pick up background tabs.
- Chrome only. The EyeDropper-style native APIs this relies on (`tabCapture`, Manifest V3 service workers) aren't available in Firefox or Safari.
- The WASM module needs to be rebuilt manually with Emscripten; it isn't wired into the `npm run build` script yet.

## Status

Work in progress. The capture pipeline, WASM FFT bridge, and fallback path all work, but the build process is still two separate manual steps (Emscripten + Vite) rather than one command.
