# Developer Context & Architecture

## 1. High-Level Architecture & Intent

*   **Core Purpose:** A WebGPU-accelerated IPTV player that parses M3U playlists client-side and renders a high-performance stream selection UI and video post-processing effects.
*   **Tech Stack:**
    *   **Frontend:** Vite, TypeScript, HTML/CSS.
    *   **Graphics:** WebGPU (via `@webgpu/types`).
    *   **Video:** `hls.js` for HLS streaming, HTML5 `<video>` element.
*   **Design Patterns:**
    *   **Renderer Separation:** UI logic (`StreamSelector`, `VideoPlayer`) is decoupled from rendering logic (`StreamSelectorRenderer`, `WebGPURenderer`).
    *   **Compute/Render Pipelines:** Explicit management of WebGPU pipelines for both UI layout (compute) and rendering.
    *   **Atlas Management:** Custom text atlas generation for efficient text rendering in WebGPU.

## 2. Feature Map

*   **M3U Playlist Parsing:**
    *   **Entry:** `web/src/m3u-parser.ts` -> `fetchGroups()`
    *   **Details:** Fetches `https://iptv-org.github.io/iptv/index.m3u` (or custom URL), parses EXTINF tags, and groups streams by `group-title`.
*   **Stream Selection UI:**
    *   **Entry:** `web/src/stream-selector.ts` (Logic) & `web/src/stream-selector-renderer.ts` (WebGPU)
    *   **Details:** A custom WebGPU renderer that handles scrolling, layout, and rendering of stream lists. Uses a compute shader for visibility culling and positioning.
*   **Video Playback & Effects:**
    *   **Entry:** `web/src/video-player.ts`
    *   **Details:** Wrapper around `hls.js`. Optionally pipes video frames into a WebGPU pipeline (`web/src/webgpu-renderer.ts`) for real-time brightness, contrast, and saturation adjustments using compute shaders.

## 3. Complexity Hotspots

*   **WebGPU Resource Management (`webgpu-renderer.ts`, `stream-selector-renderer.ts`):**
    *   **Why:** Manual management of `GPUTexture`, `GPUBuffer`, and `GPUBindGroup` is required. Resources must be destroyed and recreated when canvas dimensions change or contexts are lost.
    *   **Agent Note:** Ensure `destroy()` methods clean up all GPU resources to prevent memory leaks. Watch out for race conditions when resizing—textures might be destroyed before the next render frame uses them.
*   **Video Texture Synchronization (`webgpu-renderer.ts`):**
    *   **Why:** `copyExternalImageToTexture` copies the current video frame to a GPU texture. This must be timed correctly (e.g., `requestAnimationFrame`) and requires the video element to be ready (`readyState >= 2`).
    *   **Agent Note:** Browser support for `copyExternalImageToTexture` varies. Ensure the video source is CORS-compliant if using external URLs, or the texture copy will fail.
*   **Text Atlas Rendering (`text-atlas.ts`):**
    *   **Why:** To render text in WebGPU, the app generates a dynamic texture atlas using an `OffscreenCanvas`.
    *   **Agent Note:** The atlas has a fixed size (`2048x4096`) and a fixed item height (`40px`). It can only hold ~100 unique strings before overflowing. The current implementation creates a new atlas for every filtered list, but if the list > 100 items, visual artifacts (reused slots) will occur.

## 4. Inherent Limitations & "Here be Dragons"

*   **Atlas Overflow Bug:**
    *   `text-atlas.ts` has a capacity of ~102 items (`4096 / 40`).
    *   `stream-selector.ts` slices the display list to 200 items.
    *   **Result:** The bottom ~100 items in the list will display incorrect or corrupted text (reusing the last slot). **Do not increase the list limit without fixing the atlas pagination logic.**
*   **WebGPU Dependency:**
    *   The app *requires* a browser with WebGPU support. While there is a check (`isSupported`), the core "cool" features (custom UI, effects) are disabled without it.
*   **CORS Restrictions:**
    *   The default playlist `iptv-org.github.io` supports CORS. If the user loads a custom playlist from a server without CORS headers, the `fetch` in `m3u-parser.ts` will fail.
*   **Flag Atlas:**
    *   `text-atlas.ts` attempts to load `/flags.png`. If missing, it falls back to a 1x1 white texture. The flag mapping is hardcoded to a specific list of countries.

## 5. Dependency Graph & Key Flows

1.  **Initialization:**
    *   `main.ts` -> checks WebGPU -> inits `VideoPlayer` -> inits `StreamSelector`.
2.  **Data Loading:**
    *   `StreamSelector` -> `m3u-parser.fetchGroups()` -> `fetch(url)` -> `parseM3U()`.
3.  **UI Rendering:**
    *   `StreamSelector` -> filters streams -> `StreamSelectorRenderer.setStreams()`.
    *   `StreamSelectorRenderer` -> generates `TextAtlas` (CPU) -> uploads to GPU -> runs Compute Shader (layout) -> runs Render Shader (draw).
4.  **Playback:**
    *   User clicks stream -> `StreamSelector` emits event -> `VideoPlayer.loadStream()`.
    *   `VideoPlayer` -> `hls.js` loads URL -> `<video>` plays.
    *   (If WebGPU) `WebGPURenderer` -> `copyExternalImageToTexture` -> Compute Shader (effects) -> Render Shader (to canvas).
