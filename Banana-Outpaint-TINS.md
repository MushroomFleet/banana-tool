# BananaOutpaint

<!-- TINS Specification v1.0 -->
<!-- ZS:COMPLEXITY:MEDIUM -->
<!-- ZS:PLATFORM:WEB -->
<!-- ZS:LANGUAGE:JAVASCRIPT -->

## Description

BananaOutpaint is a single-file browser application that lets users outpaint (extend) any source image using the Google Gemini image-generation API. The user uploads a source image, selects a target canvas size and resolution, positions the source image within the canvas using an interactive drag preview, writes a prompt describing the scene, and submits. Gemini fills in the blank canvas area around the placed image. Results accumulate in an IndexedDB-backed gallery and can be bulk-exported as a ZIP.

No server, no build step. The entire app ships as one `.html` file with no external dependencies except JSZip (loaded from a CDN) and the Gemini API.

---

## Functionality

### UI Layout

```
+--------------------------------------------------+
| 🍌 BananaOutpaint                                |
+--------------------------------------------------+
| ⚙️ API Settings  [collapsible]                   |
|   Google Gemini API Key: [password input] [Save] |
|   Model: [select]                                |
+--------------------------------------------------+
| Source Image                                     |
|   [file input — PNG/JPEG]                        |
|   "filename.png (1280×720)"                      |
+--------------------------------------------------+
| Canvas Size: [select]   Resolution: [select]     |
+--------------------------------------------------+
| Position & Scale                                 |
|   [SVG canvas — white rect, draggable blue box]  |
|   Scale: 60%  [range slider 10–100]              |
|   Position — x: 0px, y: 0px (output pixels)     |
+--------------------------------------------------+
| Outpaint Prompt                                  |
|   [textarea]                                     |
|   [Outpaint button]                              |
|   Status: Ready. / processing… / Done! / Error   |
+--------------------------------------------------+
| Gallery (N results)    [Download All as ZIP]     |
|   [110×110 thumbnails, click to open fullsize]   |
+--------------------------------------------------+
```

### Panels & Interactions

**API Settings (collapsible `<details>`):**
- Password input for the Google Gemini API key. Persisted to `localStorage` on "Save" click. A green "Key saved." confirmation appears for 2 seconds.
- Model select. Currently one option: `gemini-3.1-flash-image-preview` (label: "Nano Banana 2"). Persisted to `localStorage`.

**Source Image:**
- File input restricted to `image/png` and `image/jpeg`.
- On load, display `"filename (NaturalW×NaturalH)"`. Store the natural dimensions and a blob URL in state.

**Canvas Size & Resolution:**
- Canvas size select: Square (1:1), Landscape 4:3, Landscape 16:9, Portrait 3:4, Portrait 9:16, Landscape 3:2, Portrait 2:3.
- Resolution select is disabled until a canvas size is chosen. Then it populates three tiers: 1K (maxDim 1024), 2K (maxDim 2048), 4K (maxDim 4096). Dimensions are computed as: if `ratioW >= ratioH`, output is `(maxDim, round(maxDim * ratioH / ratioW))`, else `(round(maxDim * ratioW / ratioH), maxDim)`.
- Changing either re-centres the source image box and re-renders the SVG preview.

**Position & Scale (SVG preview):**
- An SVG element (max display dimension 480px on longest side, aspect-matched to output canvas) renders a white rectangle representing the output canvas. A semi-transparent blue draggable box ("source image") represents the placed source.
- The box is scaled to `innerScalePercent`% of the SVG display size while preserving the source image's aspect ratio.
- **Drag**: Mouse and touch drag both supported. Clamped so the inner box never exceeds canvas bounds.
- **Scale slider**: Range 10–100 (integer). Default 60. Label shows current value. Changing the slider re-renders SVG; does not rebuild drag listeners unless the box geometry changes (use full `renderSVG()` on scale change).
- **Position label**: Displays x/y in output pixels (display coordinate ÷ scaleFactor, rounded).

**Outpaint Prompt:**
- `<textarea>` (resizable vertically). Prompt is trimmed on read.

**Outpaint button:**
- Enabled only when: image loaded, outputW/H > 0, prompt non-empty, API key non-empty, and status is not "processing".

**Status line:**
- Default: grey "Ready."
- Processing: yellow "Building composite canvas…" → "Sending to Gemini API…" → "Saving result…"
- Done: green "Done! Result added to gallery."
- Error: red "Error: {message}"

**Gallery:**
- 110×110px thumbnails (object-fit: contain, dark background). Click opens full-size in new tab. Hover highlights border in yellow.
- Count shown in heading. Populated from IndexedDB on load and after each successful outpaint.
- "Download All as ZIP" bundles all stored blobs as `outpaint_001.png`, `outpaint_002.png`, etc. via JSZip.

---

## Technical Implementation

### Architecture

Single-file HTML/CSS/JS. No module system, no framework. All logic runs in the browser. State is a plain JS object. DOM refs are captured once at startup.

**Dependencies:**
- JSZip 3.10.1 — `https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js`

### State Model

```javascript
{
  imageFile:         File | null,
  imageNaturalW:     number,            // source image px width
  imageNaturalH:     number,            // source image px height
  imageSrc:          string,            // blob URL
  canvasSizeLabel:   string,            // e.g. "Landscape 16:9"
  outputW:           number,            // target canvas width in px
  outputH:           number,            // target canvas height in px
  innerScalePercent: number,            // 10–100, default 60
  innerDisplayX:     number,            // box X in SVG display coordinates
  innerDisplayY:     number,            // box Y in SVG display coordinates
  prompt:            string,
  apiKey:            string,            // from localStorage
  modelId:           string,            // from localStorage
  status:            'idle' | 'processing' | 'done' | 'error',
  statusMessage:     string
}
```

### Canvas Ratio & Resolution Helpers

```javascript
const CANVAS_RATIOS = {
  'Square':         [1, 1],
  'Landscape 4:3':  [4, 3],
  'Landscape 16:9': [16, 9],
  'Portrait 3:4':   [3, 4],
  'Portrait 9:16':  [9, 16],
  'Landscape 3:2':  [3, 2],
  'Portrait 2:3':   [2, 3],
};

const RES_TIERS = [
  { label: '1K', maxDim: 1024 },
  { label: '2K', maxDim: 2048 },
  { label: '4K', maxDim: 4096 },
];

function getResolution(ratioW, ratioH, maxDim) {
  return ratioW >= ratioH
    ? [maxDim, Math.round(maxDim * ratioH / ratioW)]
    : [Math.round(maxDim * ratioW / ratioH), maxDim];
}
```

### SVG Preview Dimensions

```javascript
const DISPLAY_MAX = 480;

function computeSVGDims() {
  const aspect = state.outputW / state.outputH;
  const svgW = aspect >= 1 ? DISPLAY_MAX : Math.round(DISPLAY_MAX * aspect);
  const svgH = aspect >= 1 ? Math.round(DISPLAY_MAX / aspect) : DISPLAY_MAX;
  return { svgW, svgH, scaleFactor: svgW / state.outputW };
}

function computeInnerBox(svgW, svgH) {
  const imageAspect = state.imageNaturalW / state.imageNaturalH;
  const fitsWide = imageAspect >= svgW / svgH;
  const innerW = fitsWide
    ? svgW * (state.innerScalePercent / 100)
    : (svgH * (state.innerScalePercent / 100)) * imageAspect;
  const innerH = fitsWide
    ? innerW / imageAspect
    : svgH * (state.innerScalePercent / 100);
  return { innerW, innerH };
}

function clampPos(x, y, innerW, innerH, svgW, svgH) {
  return {
    x: Math.min(Math.max(x, 0), svgW - innerW),
    y: Math.min(Math.max(y, 0), svgH - innerH),
  };
}
```

**SVG render strategy:** `renderSVG()` does a full `svgContainer.innerHTML` rebuild (used on structural changes: canvas resize, scale change, new image). `updateInnerBoxDOM()` updates only the `x`/`y` SVG attributes in-place during drag (no DOM rebuild).

### Composite Canvas

```javascript
async function buildCompositeCanvas() {
  // Create output canvas at full target resolution
  const canvas = document.createElement('canvas');
  canvas.width = state.outputW;
  canvas.height = state.outputH;
  const ctx = canvas.getContext('2d');
  ctx.fillStyle = '#ffffff';
  ctx.fillRect(0, 0, state.outputW, state.outputH);

  // Convert display-coordinate position to output-pixel position
  const { svgW, svgH, scaleFactor } = computeSVGDims();
  const { innerW, innerH } = computeInnerBox(svgW, svgH);
  const { x: cx, y: cy } = clampPos(state.innerDisplayX, state.innerDisplayY, innerW, innerH, svgW, svgH);

  const destX = Math.round(cx / scaleFactor);
  const destY = Math.round(cy / scaleFactor);
  const destW = Math.round(innerW / scaleFactor);
  const destH = Math.round(innerH / scaleFactor);

  // Draw source image at computed destination rect
  const img = new Image();
  img.src = state.imageSrc;
  await new Promise(resolve => { img.onload = resolve; });
  ctx.drawImage(img, destX, destY, destW, destH);
  return canvas;
}
```

### Gemini API Call

Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/{modelId}:generateContent?key={apiKey}`

Request body:
```json
{
  "contents": [{
    "parts": [
      { "text": "<user prompt>" },
      { "inline_data": { "mime_type": "image/png", "data": "<base64 PNG>" } }
    ]
  }],
  "generationConfig": {
    "responseModalities": ["TEXT", "IMAGE"]
  }
}
```

- Canvas is exported as PNG via `canvas.toDataURL('image/png')`, base64 portion stripped after the comma.
- Timeout: 120 seconds via `AbortController`.
- On success: find the first `part` in `candidates[0].content.parts` where `part.inline_data` exists. Decode from base64 to a `Blob`.
- Errors: HTTP non-2xx throws the `error.message` from the JSON body. No `inline_data` part throws `"API returned text only, no image."`. AbortError throws `"Request timed out."`.

### IndexedDB

Database name: `BananaOutpaintDB`, version 1.  
Object store: `outpaint_results`, keyPath `id`, autoIncrement.

Record schema:
```javascript
{
  id:        number,        // auto-incremented
  timestamp: string,        // ISO 8601
  blob:      Blob,          // image data
  prompt:    string,
  outputW:   number,
  outputH:   number
}
```

`openDB()` — opens/creates the DB. Called once on startup; result stored in `db`. Gallery refreshes automatically on open.

`saveResult(db, blob, prompt, outputW, outputH)` — writes one record.

`getAllResults(db)` — returns all records. Gallery renders them in reverse order (newest first).

`refreshGallery()` — clears `#galleryGrid`, sets count, creates `<img>` elements from `URL.createObjectURL(record.blob)`. Click handler opens blob URL in a new tab.

### ZIP Export

Uses JSZip. All records fetched from IndexedDB, each blob added as `outpaint_NNN.png` (zero-padded to 3 digits). Zip generated as a blob and downloaded via a temporary `<a>` click. Alerts if no results or if JSZip failed to load.

### Style Guide

- Background: `#1a1a2e` (deep navy)
- Panel background: `#16213e`, border: `#0f3460`
- Input background: `#0f3460`, border: `#1a5fa8`
- Accent / primary button: `#ffe066` (yellow), text `#1a1a2e`
- Save button: `#4a90d9`
- Status colours: idle/default `#aaa`, processing `#ffe066`, done `#6bffb8`, error `#ff6b6b`
- Font: `'Segoe UI', system-ui, sans-serif`
- All elements use `box-sizing: border-box`

---

## Edge Cases

- **No image uploaded**: SVG shows "upload source image" placeholder text; Outpaint button disabled.
- **No canvas size selected**: Resolution select shows "— pick size first —" and is disabled; outputW/H remain 0.
- **Outpaint button guard**: disabled unless all four conditions met: image, canvas dims, prompt, API key.
- **Scale slider at 100%**: inner box fills entire SVG; clamping keeps it at (0,0).
- **Drag out of bounds**: `clampPos` hard-clamps x and y so the inner box cannot exit the SVG canvas.
- **API returns no image part**: throw descriptive error; status set to error state.
- **Request timeout (>120s)**: AbortController fires; error message "Request timed out."
- **IndexedDB unavailable**: status set to "Gallery unavailable: {message}"; outpainting still works but results are not saved.
- **ZIP with no results**: `alert('No results to export.')` shown; no download triggered.
- **JSZip CDN failure**: `alert('ZIP export unavailable (JSZip failed to load).')`.
