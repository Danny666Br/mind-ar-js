# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
npm install

# Development server (with HTTPS via basicSsl, hot reload via Vite)
npm run dev

# Watch mode for Three.js builds (auto-rebuild on src/ changes → dist-dev/)
npm run watch

# One-time development build (required for AFRAME builds — watch doesn't handle them)
npm run build-dev

# Production build → dist/
npm run build
```

There are no automated tests. Manual testing is done by opening HTML examples in `examples/` via a local HTTPS server (camera access requires HTTPS).

## Architecture

MindAR is a web AR library with two independent tracking systems: **Image Tracking** and **Face Tracking**. Each system exposes two integration layers: a raw **Three.js** API and an **AFRAME** component wrapper.

### Build outputs

| File | Description |
|------|-------------|
| `mindar-image[.prod].js` | Image tracking core (ES module) |
| `mindar-image-three[.prod].js` | Image tracking + Three.js helpers |
| `mindar-image-aframe[.prod].js` | Image tracking AFRAME components (IIFE) |
| `mindar-face[.prod].js` | Face tracking core (ES module) |
| `mindar-face-three[.prod].js` | Face tracking + Three.js helpers |
| `mindar-face-aframe[.prod].js` | Face tracking AFRAME components (IIFE) |

Dev builds go to `dist-dev/` (with sourcemaps, unminified). Prod builds go to `dist/`. `three` is an external peer dependency in module builds but is bundled into AFRAME builds.

### Image Tracking (`src/image-target/`)

- **`Controller`** — main orchestrator; manages camera input, detection/tracking lifecycle, and a `ControllerWorker` (Web Worker) for off-thread processing. Uses One Euro Filter to smooth pose output.
- **`Compiler`** / **`CompilerBase`** — compiles target images into `.mind` binary files (via `CompilerWorker`). The offline variant (`offline-compiler.js`) runs in Node.js using the `canvas` package.
- **`CropDetector`** (`detector/`) — runs feature detection using FREAK descriptors. Has two backends:
  - `webgl/` — GPU-accelerated kernels via TensorFlow.js WebGL backend
  - `cpu/` — CPU fallback using the same shader-like kernel interface (`fakeShader.js`)
- **`Tracker`** (`tracker/`) — tracks already-detected targets frame-to-frame via optical flow
- **`Estimator`** (`estimation/`) — computes pose (homography → 3D transform) using RANSAC + refinement
- **`Matcher`** (`matching/`) — hierarchical clustering + Hough transform for robust feature matching
- **`MindARThree`** (`three.js`) — high-level Three.js wrapper exposing `anchors[]`, `start()`, `stop()`, and the Three.js `scene`, `camera`, `renderer`

### Face Tracking (`src/face-target/`)

- **`Controller`** — wraps MediaPipe FaceMesh (`FaceMeshHelper`) + OpenCV-based geometry estimation (`Estimator`). Uses One Euro Filter on all 468 landmarks.
- **`FaceMeshHelper`** — loads and runs the MediaPipe `@mediapipe/tasks-vision` FaceDetector model
- **`Estimator`** (`face-geometry/`) — converts 2D MediaPipe landmarks to 3D metric space using canonical face model data (`face-data.js`)
- **`MindARThree`** (`three.js`) — high-level Three.js wrapper; face tracking mirrors by default (`shouldFaceUser = true`)

### Shared components

- **`UI`** (`src/ui/ui.js`) — loading/scanning/error overlay injected into the container
- **`OneEuroFilter`** (`src/libs/`) — smoothing filter applied to pose/landmark outputs to reduce jitter
- **`opencv-helper.js`** — lazy-loads the OpenCV WASM build; `waitCV()` returns a promise that resolves when OpenCV is ready

### Key design patterns

- **TensorFlow.js is used as a WebGL compute engine**, not for ML — the image detection kernels are written as TF.js custom ops that compile to WebGL shaders.
- **Web Workers** offload the heavy detection/compile work from the main thread: `controller.worker.js` handles match+track, `compiler.worker.js` handles `.mind` file compilation.
- The AFRAME builds (`aframe.js`) register custom AFRAME components and attach to `window.MINDAR.IMAGE` / `window.MINDAR.FACE`.

### Known Issues

- On Mac (Node v18+), `node-canvas` may fail to install. Fix: `brew install pkg-config cairo pango libpng jpeg giflib librsvg pixman`
- When using esbuild, import directly from source (`mind-ar/src/image-target/three.js`) and use `esbuild-plugin-inline-worker` to handle the inline workers.
- AFRAME builds do not work with `--watch`; run `npm run build-dev` manually after changes to AFRAME files.
