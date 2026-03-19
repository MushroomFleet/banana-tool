# Banana-Tool

<!-- TINS Specification v1.0 -->
<!-- ZS:COMPLEXITY:HIGH -->
<!-- ZS:PLATFORM:DESKTOP -->
<!-- ZS:LANGUAGE:TYPESCRIPT -->
<!-- ZS:FRAMEWORK:REACT -->
<!-- ZS:BUILD:VITE -->
<!-- ZS:DISTRIBUTION:TAURI -->

## Description

Banana-Tool is a unified desktop image-editing application that combines four AI-powered image manipulation modes — **Inpaint**, **Replace**, **EDIT**, and **Outpaint** — into a single gallery-centric interface. All modes use the Google Gemini generative image API. Users load images into a shared gallery, then launch mode-specific modal overlays from gallery card action buttons. Every result returns to the same gallery, tagged by mode, enabling grouped output filtering.

Built with Vite + TypeScript + React, packaged with Tauri 2 for portable Windows MSI distribution.

### Reference Codebases (available at build time)

| Ref ID | Path (relative to project root) | Contains |
|--------|--------------------------------|----------|
| `REF:INPAINT` | `BananaPro-Inpainting-dev/` | Complete React/TS inpainting app — canvas mask editor, Gemini API service, IndexedDB gallery, brush system, multi-mode (single + reference image). |
| `REF:REPLACE` | `djz-cropreplacer-dev/` | Complete vanilla JS crop-and-replace app — crop box selection, 1MP upscale pipeline, colour matching (RGB + LAB), edge blending, patch compositing, IndexedDB galleries. |
| `REF:OUTPAINT` | `Banana-Outpaint-TINS.md` | TINS spec for outpainting — SVG position preview, canvas ratio/resolution helpers, composite canvas builder, Gemini API call pattern. |

Implementors **MUST** read the reference codebases before writing code. They contain battle-tested logic that should be adapted (not copy-pasted) into the unified TypeScript/React architecture. Specific reuse points are called out with `→ REF:` annotations throughout this document.

---

## Functionality

### Application Modes

| Mode | Purpose | Input | Key Controls | Source |
|------|---------|-------|-------------|--------|
| **Inpaint** | Paint a mask on an image; AI fills masked region naturally or blends a reference image into it | Image + painted mask + optional Image 2 | Brush size/hardness/overlay, draw/erase toggle, clear/invert mask, single/multi toggle | `REF:INPAINT` |
| **Replace** | Crop a rectangular region; AI replaces its content based on a prompt | Image + crop box selection | Crop box draw/drag, strength slider, colour match toggle (RGB/LAB), edge blend slider | `REF:REPLACE` |
| **EDIT** | Send a whole image (+ optional reference image) with a text prompt; AI returns an edited version | Image + optional Image 2 | Strength (temperature) slider, resolution selector | New — simplest mode |
| **Outpaint** | Extend an image onto a larger canvas; AI fills the surrounding blank area | Image + canvas size/ratio + position/scale | Canvas ratio select, scale slider, SVG position preview | `REF:OUTPAINT` |

### UI Layout

```
+-------------------------------------------------------------+
| 🍌 Banana-Tool                          [Settings ⚙️]       |
+-------------------------------------------------------------+
| Filter: [All] [Inpaint] [Replace] [EDIT] [Outpaint] [Input] |
+-------------------------------------------------------------+
| Gallery Grid                                                 |
|  +--------+ +--------+ +--------+ +--------+ +--------+     |
|  | thumb  | | thumb  | | thumb  | | thumb  | | thumb  |     |
|  | mode   | | mode   | | mode   | | mode   | | mode   |     |
|  | tag    | | tag    | | tag    | | tag    | | tag    |     |
|  |[I][R][E][O]|[I][R][E][O]| ...                            |
|  +--------+ +--------+ +--------+ +--------+ +--------+     |
|  (scrollable grid, responsive columns)                       |
+-------------------------------------------------------------+
| [Load Image] [Download Selected] [Delete Selected] [ZIP All]|
+-------------------------------------------------------------+
```

#### Header Bar
- App title "Banana-Tool" with banana emoji.
- Settings gear button opens `SettingsModal`.

#### Filter Bar
- Horizontal row of toggle buttons: `All`, `Inpaint`, `Replace`, `EDIT`, `Outpaint`, `Input`.
- `All` shows every gallery item. Other buttons filter by `item.mode` tag.
- `Input` shows only user-uploaded source images (mode = `"input"`).
- Active filter button uses accent colour; others are muted.
- Default filter on launch: `All`.

#### Gallery Grid
- Responsive CSS grid: `grid-template-columns: repeat(auto-fill, minmax(180px, 1fr))`.
- Each card is a `GalleryCard` component (see below).
- Sorted newest-first by `createdAt`.
- Empty state: centred text "Drop images here or click Load Image".

#### Gallery Card

```
+---------------------------+
| [checkbox]       [mode]   |
|                           |
|      thumbnail            |
|     (object-fit:cover)    |
|                           |
| filename.png              |
| 1280×720 · 2.4 MB        |
|                           |
| [Inpaint] [Replace]      |
| [EDIT]    [Outpaint]      |
+---------------------------+
```

- **Checkbox**: top-left, for bulk selection (download/delete).
- **Mode badge**: top-right pill showing the mode tag in mode colour.
- **Thumbnail**: 180×180 CSS, `object-fit: cover`, dark background. Double-click opens Lightbox.
- **Metadata line**: filename, dimensions, file size.
- **Action buttons**: Four small icon-buttons. Clicking any button opens that mode's modal overlay with this image pre-loaded as the primary input.

Mode badge colours:
| Mode | Colour | Label |
|------|--------|-------|
| `input` | `#8e8e93` (grey) | INPUT |
| `inpaint` | `#f5c542` (yellow) | INPAINT |
| `replace` | `#63B3ED` (blue) | REPLACE |
| `edit` | `#6bffb8` (green) | EDIT |
| `outpaint` | `#e09eff` (purple) | OUTPAINT |

#### Bottom Toolbar
- **Load Image**: file picker (`image/png, image/jpeg, image/webp`). Loaded images are saved to gallery with `mode: "input"`.
- **Download Selected**: downloads all checked images as individual files (or single file if one selected).
- **Delete Selected**: confirmation dialog, then removes checked items from IndexedDB.
- **ZIP All**: exports all gallery items (or filtered subset) as `banana-tool-export.zip` using JSZip.
- Drag-and-drop onto the gallery area also loads images.

#### Lightbox
- Full-viewport overlay on double-click any thumbnail.
- Shows image at native resolution, scrollable/zoomable.
- Close on click-outside, Escape, or X button.
- Nav arrows if multiple images in current filter.
- → Adapt from `REF:INPAINT` `Lightbox.tsx`.

---

### Settings Modal

Opened from header gear button. Persisted to `localStorage`.

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| **API Key** | password input | `""` | Google Gemini API key |
| **Model** | select | `gemini-3.1-flash-image-preview` | Options: `gemini-3.1-flash-image-preview` ("Nano Banana 2"), `gemini-3-pro-image-preview` ("Nano Banana Pro") |
| **Default Resolution** | button group | `1K` | `1K` / `2K` / `4K` — used as initial value in all modals |
| **Output Format** | radio | `PNG` | `PNG` / `JPEG` |
| **JPEG Quality** | slider | `0.92` | `0.1–1.0`, shown only if JPEG selected |

→ Adapt settings persistence from `REF:INPAINT` `src/lib/settings.ts`.

---

### Modal Overlays — Shared Structure

All four mode modals share a common shell:

```
+----------------------------------------------------------+
| [X close]              MODE NAME                         |
+----------------------------------------------------------+
| +------------------+  +-------------------------------+  |
| | Primary Image    |  | Mode-specific controls        |  |
| | (preview)        |  |                               |  |
| |                  |  | Prompt: [textarea]            |  |
| |                  |  |                               |  |
| |                  |  | Resolution: [1K] [2K] [4K]   |  |
| |                  |  |                               |  |
| |                  |  | [mode-specific sliders]       |  |
| |                  |  |                               |  |
| +------------------+  | [Generate] [Cancel]           |  |
|                       +-------------------------------+  |
+----------------------------------------------------------+
| Result Preview (appears after generation)                |
| [Save to Gallery] [Regenerate] [Discard]                 |
+----------------------------------------------------------+
```

- **Backdrop**: semi-transparent black overlay. Click-outside does NOT close (prevents accidental loss).
- **Close button** (X): top-right. If processing, shows "Cancel generation?" confirm.
- **Primary Image preview**: left column, max 400px wide, aspect-ratio preserved.
- **Prompt textarea**: shared across all modes, resizable vertically.
- **Resolution buttons**: `1K` / `2K` / `4K`, maps to Gemini `imageConfig.imageSize`.
- **Generate button**: disabled until all required inputs present. Shows spinner during processing.
- **Result Preview**: below the main area. Shows the generated image. User can Save, Regenerate (same settings), or Discard.
- **Status line**: bottom of modal. Same colour coding as `REF:OUTPAINT` (grey idle, yellow processing, green done, red error).

---

### Inpaint Modal

Opens when user clicks "Inpaint" on a gallery card.

#### Layout

Left column: **Canvas Editor** — the selected image with paintable mask overlay.
Right column: controls.

#### Controls

| Control | Type | Range | Default | Maps to |
|---------|------|-------|---------|---------|
| **Mode toggle** | button pair | `Single` / `Multi` | `Single` | Determines if Image 2 is used |
| **Image 2** (multi only) | file picker + thumbnail | — | `null` | Reference image for blending |
| **Brush Size** | slider | 2–120 px | 30 | Brush diameter |
| **Hardness** | slider | 0–100% | 80% | Brush edge gradient (0 = soft, 100 = hard) |
| **Overlay Opacity** | slider | 10–100% | 50% | Red mask preview opacity |
| **Draw / Erase** | toggle | — | Draw | Paints white (mask) or black (unmask) on mask canvas |
| **Clear Mask** | button | — | — | Fills mask canvas with black |
| **Invert Mask** | button | — | — | Inverts all mask pixel values |
| **Prompt** | textarea | — | `""` | Inpaint instruction text |
| **Resolution** | button group | 1K/2K/4K | from settings | Output resolution |

#### Canvas Editor

Two overlaid HTML5 canvases at the image's natural resolution:
1. **Display canvas** — renders original image + red-tinted mask overlay.
2. **Mask canvas** — hidden, captures mouse/touch input, stores black/white mask data.

→ Adapt directly from `REF:INPAINT` `src/hooks/useMaskEditor.ts` and `src/components/CanvasEditor.tsx`.

**Brush system**: radial gradient for soft edges. `hardness` controls gradient falloff. Drawing interpolates between mouse positions for smooth strokes. Supports mouse and touch input. Coordinate scaling accounts for CSS display size vs. canvas natural size.

#### API Call

**Single mode**: image + mask + prompt → Gemini.
- Default prompt (if empty): `"Remove the masked area and fill it naturally with surrounding background"`.
- Parts: `[text, image_inline, mask_inline]`.

**Multi mode**: image1 + mask + image2 + prompt → Gemini.
- Default prompt (if empty): `"Seamlessly blend the subject/content from Image 2 into the masked area of Image 1, matching lighting, perspective, and style"`.
- Parts: `[text, image1_inline, mask_inline, image2_inline]`.

→ Adapt from `REF:INPAINT` `src/lib/gemini.ts` functions `inpaint()` and `inpaintMulti()`.

#### Generation Config

```json
{
  "responseModalities": ["image", "text"],
  "temperature": 1.0,
  "imageConfig": { "imageSize": "<1K|2K|4K>" }
}
```

---

### Replace Modal

Opens when user clicks "Replace" on a gallery card.

#### Layout

Left column: **Crop Canvas** — the selected image with interactive crop box.
Right column: controls.

#### Crop Canvas

Full-resolution HTML5 canvas displaying the source image. User draws a rectangular selection box.

**States**:
- `idle`: no box drawn. Cursor is crosshair.
- `drawing`: mouse down and dragging. Dashed blue (`#63B3ED`) outline, 45% black overlay outside box, dimension tooltip.
- `confirmed`: mouse up. Solid white 2px border. Box draggable by clicking inside. Minimum size: 64×64 px.

→ Adapt crop box logic from `REF:REPLACE` `src/main.js` (box drawing, overlay rendering, coordinate normalization, drag-to-reposition).

**Coordinate mapping**: `toImageSpace(canvas, clientX, clientY)` converts mouse coordinates to canvas pixel coordinates, accounting for CSS scaling.

#### Controls

| Control | Type | Range | Default | Maps to |
|---------|------|-------|---------|---------|
| **Image 2** (optional) | file picker + thumbnail | — | `null` | Reference image, fill-and-cropped to match crop dimensions |
| **Strength** | slider | 0.0–1.0 | 0.75 | Gemini `temperature` parameter |
| **Colour Match** | toggle | ON/OFF | ON | Apply colour correction to patch |
| **LAB Mode** | toggle | ON/OFF | OFF | Use CIELAB instead of RGB for colour matching |
| **Edge Blend** | slider | 0–32 px | 8 | Feather radius on patch edges |
| **Prompt** | textarea | — | `""` | EDIT instruction for the cropped region |
| **Resolution** | button group | 1K/2K/4K | from settings | Gemini `imageSize` |
| **Clear Box** | button | — | — | Reset crop state to idle |

#### Processing Pipeline

1. **Extract crop** from source image at `{x, y, w, h}` in image space.
2. **Upscale to 1MP**: `scale = sqrt(1_000_000 / (w * h))`. Resample with high-quality smoothing. → `REF:REPLACE` `image-utils.js`.
3. **Send to Gemini**: upscaled crop (+ optional Image 2) + prompt + resolution + temperature.
4. **Receive patch** at API resolution.
5. **Colour match** (if enabled):
   - RGB mode: compute mean + std-dev per channel for patch and original crop, apply transfer formula `((val - patchMean) / patchStd) * refStd + refMean`, clamp to [0,255].
   - LAB mode: convert RGB → sRGB linear → XYZ → LAB, apply same formula in LAB space, convert back. → `REF:REPLACE` `colour-lab.js`.
6. **Downscale patch** from API output to original crop dimensions.
7. **Edge blend** (if > 0px): create mask canvas, draw 4 linear gradients on edges (inward fade), apply via `destination-in` composite. → `REF:REPLACE` `image-utils.js`.
8. **Composite**: draw full source image on output canvas, then draw blended patch at original `{x, y, w, h}`.
9. **Result** shown in modal result preview.

#### API Call

Same endpoint as Inpaint. Parts: `[text, crop_inline, optional_image2_inline]`.

```json
{
  "responseModalities": ["image", "text"],
  "temperature": "<strength 0.0-1.0>",
  "imageConfig": { "imageSize": "<1K|2K|4K>" }
}
```

---

### EDIT Modal

Opens when user clicks "EDIT" on a gallery card. Simplest mode — no canvas interaction.

#### Layout

Left column: **Image Preview** — read-only display of the selected image.
Right column: controls.

#### Controls

| Control | Type | Range | Default | Maps to |
|---------|------|-------|---------|---------|
| **Image 2** (optional) | file picker + thumbnail | — | `null` | Reference image sent alongside |
| **Strength** | slider | 0.0–1.0 | 0.75 | Gemini `temperature` |
| **Prompt** | textarea | — | `""` | Edit instruction |
| **Resolution** | button group | 1K/2K/4K | from settings | Gemini `imageSize` |

#### API Call

Parts: `[text, image_inline, optional_image2_inline]`.

Default prompt (if empty): `"Enhance and improve this image"`.

```json
{
  "responseModalities": ["image", "text"],
  "temperature": "<strength>",
  "imageConfig": { "imageSize": "<1K|2K|4K>" }
}
```

---

### Outpaint Modal

Opens when user clicks "Outpaint" on a gallery card.

#### Layout

Left column: **SVG Position Preview** — interactive placement of source image within target canvas.
Right column: controls.

#### SVG Preview

- SVG element, max display dimension 480px on longest side, aspect-matched to chosen canvas ratio.
- White rectangle = output canvas area.
- Semi-transparent blue draggable box = source image placement.
- Box preserves source image aspect ratio.
- Mouse and touch drag supported, clamped to canvas bounds.
- Position label shows x/y in output pixels.

→ Adapt from `REF:OUTPAINT` SVG preview spec (computeSVGDims, computeInnerBox, clampPos, drag handlers).

#### Controls

| Control | Type | Range | Default | Maps to |
|---------|------|-------|---------|---------|
| **Canvas Ratio** | select | Square 1:1, Landscape 4:3, Landscape 16:9, Portrait 3:4, Portrait 9:16, Landscape 3:2, Portrait 2:3 | Square 1:1 | Determines output aspect ratio |
| **Scale** | slider | 10–100% | 60 | Source image scale within canvas |
| **Prompt** | textarea | — | `""` | Scene description for outpainted area |
| **Resolution** | button group | 1K/2K/4K | from settings | Output resolution tier |

**Resolution computation**: → `REF:OUTPAINT` `getResolution(ratioW, ratioH, maxDim)`.

```typescript
const CANVAS_RATIOS: Record<string, [number, number]> = {
  'Square 1:1':       [1, 1],
  'Landscape 4:3':    [4, 3],
  'Landscape 16:9':   [16, 9],
  'Portrait 3:4':     [3, 4],
  'Portrait 9:16':    [9, 16],
  'Landscape 3:2':    [3, 2],
  'Portrait 2:3':     [2, 3],
};

function getResolution(ratioW: number, ratioH: number, maxDim: number): [number, number] {
  return ratioW >= ratioH
    ? [maxDim, Math.round(maxDim * ratioH / ratioW)]
    : [Math.round(maxDim * ratioW / ratioH), maxDim];
}
```

#### Composite Canvas Build

Before API call, build a full-resolution composite canvas:

1. Create canvas at `outputW × outputH`.
2. Fill with white (`#ffffff`).
3. Convert SVG display coordinates to output pixel coordinates via `scaleFactor = svgW / outputW`.
4. Draw source image at computed destination rect `(destX, destY, destW, destH)`.
5. Export as PNG base64.
6. Send composite + prompt to Gemini.

→ Adapt `buildCompositeCanvas()` from `REF:OUTPAINT`.

#### API Call

Parts: `[text, composite_inline]`.

No `imageConfig.imageSize` — Gemini receives the composite at final target resolution and returns the outpainted result at that resolution.

```json
{
  "contents": [{ "parts": [
    { "text": "<prompt>" },
    { "inlineData": { "mimeType": "image/png", "data": "<composite base64>" } }
  ]}],
  "generationConfig": {
    "responseModalities": ["image", "text"]
  }
}
```

---

## Technical Implementation

### Architecture

```
src/
├── main.tsx                    # React DOM entry
├── App.tsx                     # Root component: gallery + modal routing
├── App.css                     # Global layout styles
├── index.css                   # CSS variables, theme, reset
├── types.ts                    # All shared TypeScript types
├── lib/
│   ├── gemini.ts               # Gemini API client (all modes)
│   ├── galleryDb.ts            # IndexedDB abstraction
│   ├── settings.ts             # localStorage settings persistence
│   ├── imageUtils.ts           # Canvas helpers, upscale, downscale, composite
│   ├── colourLab.ts            # LAB colour space conversion & matching
│   └── filenaming.ts           # Filename generation logic
├── hooks/
│   ├── useMaskEditor.ts        # Canvas mask paint state (Inpaint)
│   ├── useCropBox.ts           # Crop box selection state (Replace)
│   └── useOutpaintLayout.ts    # SVG preview state (Outpaint)
├── components/
│   ├── Header.tsx              # App title + settings button
│   ├── FilterBar.tsx           # Mode filter toggle buttons
│   ├── GalleryGrid.tsx         # Responsive image grid
│   ├── GalleryCard.tsx         # Single card: thumb, meta, action buttons
│   ├── GalleryToolbar.tsx      # Load, download, delete, ZIP buttons
│   ├── Lightbox.tsx            # Full-size image overlay
│   ├── SettingsModal.tsx       # API key, model, defaults
│   ├── ModalShell.tsx          # Shared modal backdrop + layout shell
│   ├── PromptInput.tsx         # Reusable prompt textarea
│   ├── ResolutionPicker.tsx    # Reusable 1K/2K/4K button group
│   ├── Image2Picker.tsx        # Reusable optional Image 2 upload
│   ├── StatusLine.tsx          # Processing status display
│   ├── ResultPreview.tsx       # Post-generation result display + actions
│   ├── InpaintModal.tsx        # Inpaint mode modal
│   ├── CanvasEditor.tsx        # Mask painting canvas (used by InpaintModal)
│   ├── BrushToolbar.tsx        # Brush controls (used by InpaintModal)
│   ├── ReplaceModal.tsx        # Replace mode modal
│   ├── CropCanvas.tsx          # Crop box canvas (used by ReplaceModal)
│   ├── EditModal.tsx           # EDIT mode modal
│   └── OutpaintModal.tsx       # Outpaint mode modal
└── assets/
    └── (app icons if needed)
```

### Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | React | 19.x |
| Language | TypeScript | 5.x |
| Build | Vite | 7.x |
| Desktop | Tauri | 2.x |
| Storage | IndexedDB | native |
| Settings | localStorage | native |
| ZIP export | JSZip | 3.10.x (npm) |

### Project Initialization

```bash
npm create vite@latest banana-tool -- --template react-ts
cd banana-tool
npm install jszip
npm install -D @tauri-apps/cli@latest
npx tauri init
```

Tauri config: set `identifier` to `"com.bananatool.app"`, `title` to `"Banana-Tool"`, `width` to `1280`, `height` to `800`.

---

### Data Models

#### `GalleryItem`

```typescript
interface GalleryItem {
  id: string;                // crypto.randomUUID()
  filename: string;          // display filename (inherited + timestamp)
  dataUrl: string;           // data:image/png;base64,... (for small items) OR blob reference
  blob: Blob;                // raw image blob stored in IndexedDB
  mode: GalleryMode;         // tag for filtering
  prompt: string;            // prompt used (empty for input images)
  parentId: string | null;   // id of source image (null for user uploads)
  parentFilename: string;    // original source filename
  width: number;             // natural image width
  height: number;            // natural image height
  createdAt: number;         // Date.now() timestamp
  metadata: ModeMetadata;    // mode-specific extra data
}

type GalleryMode = 'input' | 'inpaint' | 'replace' | 'edit' | 'outpaint';

type ModeMetadata =
  | { type: 'input' }
  | { type: 'inpaint'; inpaintMode: 'single' | 'multi'; resolution: OutputResolution }
  | { type: 'replace'; cropRect: CropRect; strength: number; colourMatch: boolean; labMode: boolean; edgeBlend: number; resolution: OutputResolution }
  | { type: 'edit'; strength: number; resolution: OutputResolution }
  | { type: 'outpaint'; canvasRatio: string; scale: number; position: { x: number; y: number }; outputW: number; outputH: number };
```

#### `AppSettings`

```typescript
interface AppSettings {
  apiKey: string;
  modelName: string;
  defaultResolution: OutputResolution;
  outputFormat: 'png' | 'jpeg';
  jpegQuality: number;
}

type OutputResolution = '1K' | '2K' | '4K';
```

#### `BrushSettings`

```typescript
interface BrushSettings {
  size: number;       // 2-120
  hardness: number;   // 0-1
  opacity: number;    // 0.1-1 (overlay visibility)
}

type DrawMode = 'draw' | 'erase';
```

#### `CropRect`

```typescript
interface CropRect {
  x: number;
  y: number;
  w: number;
  h: number;
}

type BoxState = 'idle' | 'drawing' | 'confirmed';
```

#### `ProcessingState`

```typescript
interface ProcessingState {
  status: 'idle' | 'processing' | 'done' | 'error';
  message?: string;
}
```

---

### IndexedDB Schema

Database name: `BananaToolDB`, version `1`.

Object store: `gallery`, keyPath: `id`.

Indexes:
- `createdAt` — for newest-first sorting.
- `mode` — for filtered queries.
- `parentId` — for finding derivatives.

```typescript
// lib/galleryDb.ts

const DB_NAME = 'BananaToolDB';
const DB_VERSION = 1;
const STORE_NAME = 'gallery';

function openDb(): Promise<IDBDatabase> {
  return new Promise((resolve, reject) => {
    const req = indexedDB.open(DB_NAME, DB_VERSION);
    req.onupgradeneeded = () => {
      const db = req.result;
      if (!db.objectStoreNames.contains(STORE_NAME)) {
        const store = db.createObjectStore(STORE_NAME, { keyPath: 'id' });
        store.createIndex('createdAt', 'createdAt');
        store.createIndex('mode', 'mode');
        store.createIndex('parentId', 'parentId');
      }
    };
    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
  });
}

async function saveItem(item: GalleryItem): Promise<void> { /* put into store */ }
async function getAllItems(): Promise<GalleryItem[]> { /* getAll, sort by createdAt desc */ }
async function getItemsByMode(mode: GalleryMode): Promise<GalleryItem[]> { /* index query */ }
async function deleteItems(ids: string[]): Promise<void> { /* delete each id */ }
async function getItem(id: string): Promise<GalleryItem | undefined> { /* get by key */ }
```

→ Adapt structure from `REF:INPAINT` `src/lib/galleryDb.ts`, extending the schema with `mode`, `parentId`, `metadata`, `blob`, and `filename` fields.

---

### Filename Generation

```typescript
// lib/filenaming.ts

function generateFilename(parentFilename: string, mode: GalleryMode): string {
  const baseName = parentFilename.replace(/\.[^.]+$/, ''); // strip extension
  const timestamp = Date.now();
  return `${baseName}_${mode}_${timestamp}.png`;
}
```

All output images are saved as PNG regardless of the settings `outputFormat` — the format setting applies only to manual downloads/exports. Internal storage is always PNG for lossless quality.

**Examples**:
- Parent: `photo.jpg` → Inpaint result: `photo_inpaint_1710700800000.png`
- Parent: `photo_inpaint_1710700800000.png` → EDIT result: `photo_inpaint_1710700800000_edit_1710700900000.png`

---

### Gemini API Client

```typescript
// lib/gemini.ts

const API_BASE = 'https://generativelanguage.googleapis.com/v1beta/models';

interface GeminiImagePart {
  inlineData: { mimeType: string; data: string };
}

interface GeminiTextPart {
  text: string;
}

type GeminiPart = GeminiImagePart | GeminiTextPart;

interface GeminiRequest {
  contents: [{ role: 'user'; parts: GeminiPart[] }];
  generationConfig: {
    responseModalities: ['image', 'text'];
    temperature?: number;
    imageConfig?: { imageSize: string };
  };
}

async function callGemini(
  apiKey: string,
  modelName: string,
  parts: GeminiPart[],
  options?: { temperature?: number; imageSize?: OutputResolution }
): Promise<string> {
  // Returns data URL: "data:image/<mime>;base64,<data>"
  const url = `${API_BASE}/${modelName}:generateContent?key=${apiKey}`;

  const body: GeminiRequest = {
    contents: [{ role: 'user', parts }],
    generationConfig: {
      responseModalities: ['image', 'text'],
      ...(options?.temperature !== undefined && { temperature: options.temperature }),
      ...(options?.imageSize && { imageConfig: { imageSize: options.imageSize } }),
    },
  };

  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 120_000);

  try {
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
      signal: controller.signal,
    });

    if (!res.ok) {
      const err = await res.json();
      throw new Error(err.error?.message || `HTTP ${res.status}`);
    }

    const data = await res.json();
    const candidates = data.candidates?.[0]?.content?.parts ?? [];
    for (const part of candidates) {
      if (part.inlineData?.mimeType?.startsWith('image/')) {
        return `data:${part.inlineData.mimeType};base64,${part.inlineData.data}`;
      }
    }
    throw new Error('API returned text only, no image.');
  } catch (e: any) {
    if (e.name === 'AbortError') throw new Error('Request timed out (120s).');
    throw e;
  } finally {
    clearTimeout(timeout);
  }
}
```

#### Mode-Specific API Wrappers

```typescript
async function geminiInpaint(
  apiKey: string, model: string,
  imageB64: string, maskB64: string,
  prompt: string, resolution: OutputResolution
): Promise<string> {
  const text = prompt || 'Remove the masked area and fill it naturally with surrounding background';
  return callGemini(apiKey, model, [
    { text },
    { inlineData: { mimeType: 'image/png', data: imageB64 } },
    { inlineData: { mimeType: 'image/png', data: maskB64 } },
  ], { temperature: 1.0, imageSize: resolution });
}

async function geminiInpaintMulti(
  apiKey: string, model: string,
  image1B64: string, maskB64: string, image2B64: string,
  prompt: string, resolution: OutputResolution
): Promise<string> {
  const text = prompt || 'Seamlessly blend the subject/content from Image 2 into the masked area of Image 1, matching lighting, perspective, and style';
  return callGemini(apiKey, model, [
    { text },
    { inlineData: { mimeType: 'image/png', data: image1B64 } },
    { inlineData: { mimeType: 'image/png', data: maskB64 } },
    { inlineData: { mimeType: 'image/png', data: image2B64 } },
  ], { temperature: 1.0, imageSize: resolution });
}

async function geminiReplace(
  apiKey: string, model: string,
  cropB64: string, prompt: string,
  resolution: OutputResolution, strength: number,
  image2B64?: string
): Promise<string> {
  const parts: GeminiPart[] = [
    { text: prompt || 'Edit this image region as instructed' },
    { inlineData: { mimeType: 'image/png', data: cropB64 } },
  ];
  if (image2B64) {
    parts.push({ inlineData: { mimeType: 'image/png', data: image2B64 } });
  }
  return callGemini(apiKey, model, parts, { temperature: strength, imageSize: resolution });
}

async function geminiEdit(
  apiKey: string, model: string,
  imageB64: string, prompt: string,
  resolution: OutputResolution, strength: number,
  image2B64?: string
): Promise<string> {
  const parts: GeminiPart[] = [
    { text: prompt || 'Enhance and improve this image' },
    { inlineData: { mimeType: 'image/png', data: imageB64 } },
  ];
  if (image2B64) {
    parts.push({ inlineData: { mimeType: 'image/png', data: image2B64 } });
  }
  return callGemini(apiKey, model, parts, { temperature: strength, imageSize: resolution });
}

async function geminiOutpaint(
  apiKey: string, model: string,
  compositeB64: string, prompt: string
): Promise<string> {
  return callGemini(apiKey, model, [
    { text: prompt },
    { inlineData: { mimeType: 'image/png', data: compositeB64 } },
  ]);
  // No imageSize — composite is already at target resolution
}
```

---

### Image Utility Functions

```typescript
// lib/imageUtils.ts

/** Convert a File or Blob to base64 string (without data URL prefix) */
async function blobToBase64(blob: Blob): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      const result = reader.result as string;
      resolve(result.split(',')[1]);
    };
    reader.onerror = reject;
    reader.readAsDataURL(blob);
  });
}

/** Convert a data URL to a Blob */
function dataUrlToBlob(dataUrl: string): Blob {
  const [header, b64] = dataUrl.split(',');
  const mime = header.match(/:(.*?);/)?.[1] || 'image/png';
  const bytes = atob(b64);
  const arr = new Uint8Array(bytes.length);
  for (let i = 0; i < bytes.length; i++) arr[i] = bytes.charCodeAt(i);
  return new Blob([arr], { type: mime });
}

/** Load an Image element from a blob URL or data URL */
function loadImage(src: string): Promise<HTMLImageElement> {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => resolve(img);
    img.onerror = reject;
    img.src = src;
  });
}

/** Extract a rectangular crop from a canvas */
function extractCrop(source: HTMLCanvasElement, rect: CropRect): HTMLCanvasElement {
  const canvas = document.createElement('canvas');
  canvas.width = rect.w;
  canvas.height = rect.h;
  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(source, rect.x, rect.y, rect.w, rect.h, 0, 0, rect.w, rect.h);
  return canvas;
}

/** Upscale a canvas to target 1MP while preserving aspect ratio */
function upscaleTo1MP(source: HTMLCanvasElement): HTMLCanvasElement {
  const { width: w, height: h } = source;
  const scale = Math.sqrt(1_000_000 / (w * h));
  const targetW = Math.round(w * scale);
  const targetH = Math.round(h * scale);
  const canvas = document.createElement('canvas');
  canvas.width = targetW;
  canvas.height = targetH;
  const ctx = canvas.getContext('2d')!;
  ctx.imageSmoothingEnabled = true;
  ctx.imageSmoothingQuality = 'high';
  ctx.drawImage(source, 0, 0, targetW, targetH);
  return canvas;
}

/** Downscale a canvas to target dimensions */
function downscale(source: HTMLCanvasElement, targetW: number, targetH: number): HTMLCanvasElement {
  const canvas = document.createElement('canvas');
  canvas.width = targetW;
  canvas.height = targetH;
  const ctx = canvas.getContext('2d')!;
  ctx.imageSmoothingEnabled = true;
  ctx.imageSmoothingQuality = 'high';
  ctx.drawImage(source, 0, 0, targetW, targetH);
  return canvas;
}

/** Apply edge blend mask to a canvas (feather radius in pixels) */
function applyEdgeBlend(source: HTMLCanvasElement, radius: number): HTMLCanvasElement {
  if (radius <= 0) return source;
  const { width: w, height: h } = source;
  const canvas = document.createElement('canvas');
  canvas.width = w;
  canvas.height = h;
  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(source, 0, 0);

  // Create alpha mask with feathered edges
  const mask = document.createElement('canvas');
  mask.width = w;
  mask.height = h;
  const mCtx = mask.getContext('2d')!;
  mCtx.fillStyle = '#fff';
  mCtx.fillRect(0, 0, w, h);

  // Top edge gradient
  const gTop = mCtx.createLinearGradient(0, 0, 0, radius);
  gTop.addColorStop(0, 'rgba(255,255,255,0)');
  gTop.addColorStop(1, 'rgba(255,255,255,1)');
  mCtx.fillStyle = gTop;
  mCtx.globalCompositeOperation = 'destination-in';
  mCtx.fillRect(0, 0, w, radius);

  // Bottom edge gradient
  const gBot = mCtx.createLinearGradient(0, h, 0, h - radius);
  gBot.addColorStop(0, 'rgba(255,255,255,0)');
  gBot.addColorStop(1, 'rgba(255,255,255,1)');
  mCtx.fillRect(0, h - radius, w, radius);

  // Left edge gradient
  const gLeft = mCtx.createLinearGradient(0, 0, radius, 0);
  gLeft.addColorStop(0, 'rgba(255,255,255,0)');
  gLeft.addColorStop(1, 'rgba(255,255,255,1)');
  mCtx.fillStyle = gLeft;
  mCtx.fillRect(0, 0, radius, h);

  // Right edge gradient
  const gRight = mCtx.createLinearGradient(w, 0, w - radius, 0);
  gRight.addColorStop(0, 'rgba(255,255,255,0)');
  gRight.addColorStop(1, 'rgba(255,255,255,1)');
  mCtx.fillStyle = gRight;
  mCtx.fillRect(w - radius, 0, radius, h);

  // Apply mask to source
  ctx.globalCompositeOperation = 'destination-in';
  ctx.drawImage(mask, 0, 0);

  return canvas;
}

/** Composite a patch onto a full image at the given crop rect */
function compositePatch(
  fullImage: HTMLCanvasElement,
  patch: HTMLCanvasElement,
  rect: CropRect
): HTMLCanvasElement {
  const canvas = document.createElement('canvas');
  canvas.width = fullImage.width;
  canvas.height = fullImage.height;
  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(fullImage, 0, 0);
  ctx.drawImage(patch, rect.x, rect.y, rect.w, rect.h);
  return canvas;
}

/** Get canvas as base64 (no data URL prefix) */
function canvasToBase64(canvas: HTMLCanvasElement): string {
  return canvas.toDataURL('image/png').split(',')[1];
}

/** Get natural dimensions and file size from a File */
async function getImageInfo(file: File): Promise<{ width: number; height: number; size: number }> {
  const url = URL.createObjectURL(file);
  const img = await loadImage(url);
  URL.revokeObjectURL(url);
  return { width: img.naturalWidth, height: img.naturalHeight, size: file.size };
}
```

→ Adapt `extractCrop`, `upscaleTo1MP`, `downscale`, `applyEdgeBlend`, `compositePatch` from `REF:REPLACE` `image-utils.js`.

---

### Colour Matching

```typescript
// lib/colourLab.ts

interface ChannelStats { mean: number; std: number; }

/** Compute per-channel mean and std for an ImageData */
function rgbStats(data: ImageData): [ChannelStats, ChannelStats, ChannelStats] { /* ... */ }

/** Apply colour transfer in RGB space */
function colourMatchRGB(patch: ImageData, ref: ImageData): ImageData { /* ... */ }

/** sRGB → Linear → XYZ → LAB conversion */
function rgbToLab(r: number, g: number, b: number): [number, number, number] { /* ... */ }

/** LAB → XYZ → Linear → sRGB conversion */
function labToRgb(L: number, a: number, b: number): [number, number, number] { /* ... */ }

/** Apply colour transfer in CIELAB space */
function colourMatchLAB(patch: ImageData, ref: ImageData): ImageData { /* ... */ }
```

→ Adapt directly from `REF:REPLACE` `colour-lab.js`. Convert from vanilla JS to TypeScript with proper type annotations.

---

### Custom Hooks

#### `useMaskEditor` (Inpaint)

→ Adapt from `REF:INPAINT` `src/hooks/useMaskEditor.ts`.

Returns:
```typescript
{
  displayCanvasRef: RefObject<HTMLCanvasElement>;
  maskCanvasRef: RefObject<HTMLCanvasElement>;
  initCanvas: (img: HTMLImageElement) => void;
  paintStroke: (x: number, y: number, brush: BrushSettings, mode: DrawMode) => void;
  interpolateStroke: (from: Point, to: Point, brush: BrushSettings, mode: DrawMode) => void;
  updatePreview: (opacity: number) => void;
  clearMask: () => void;
  invertMask: () => void;
  getCoords: (e: MouseEvent) => Point;
  getTouchCoords: (e: TouchEvent) => Point;
  getMaskBase64: () => string;
  getImageBase64: () => string;
}
```

#### `useCropBox` (Replace)

→ Adapt crop box logic from `REF:REPLACE` `src/main.js`.

Returns:
```typescript
{
  canvasRef: RefObject<HTMLCanvasElement>;
  boxState: BoxState;
  cropRect: CropRect | null;
  initCanvas: (img: HTMLImageElement) => void;
  onMouseDown: (e: MouseEvent) => void;
  onMouseMove: (e: MouseEvent) => void;
  onMouseUp: (e: MouseEvent) => void;
  clearBox: () => void;
  renderOverlay: () => void;
  toImageSpace: (clientX: number, clientY: number) => Point;
}
```

#### `useOutpaintLayout` (Outpaint)

→ Adapt SVG preview logic from `REF:OUTPAINT`.

Returns:
```typescript
{
  svgContainerRef: RefObject<HTMLDivElement>;
  svgDims: { svgW: number; svgH: number; scaleFactor: number };
  innerBox: { innerW: number; innerH: number };
  position: { x: number; y: number };
  outputDims: { outputW: number; outputH: number };
  setCanvasRatio: (ratio: string) => void;
  setResolutionTier: (tier: OutputResolution) => void;
  setScale: (percent: number) => void;
  setPosition: (x: number, y: number) => void;
  renderSVG: () => void;
  buildCompositeCanvas: (imageSrc: string) => Promise<HTMLCanvasElement>;
}
```

---

### App Component State Flow

```typescript
// App.tsx — key state

const [settings, setSettings] = useState<AppSettings>(loadSettings);
const [gallery, setGallery] = useState<GalleryItem[]>([]);
const [filter, setFilter] = useState<GalleryMode | 'all'>('all');
const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
const [lightboxItem, setLightboxItem] = useState<GalleryItem | null>(null);

// Modal state
const [activeModal, setActiveModal] = useState<{
  mode: 'inpaint' | 'replace' | 'edit' | 'outpaint';
  sourceItem: GalleryItem;
} | null>(null);

// Refresh gallery from IndexedDB
const refreshGallery = useCallback(async () => {
  const db = await openDb();
  const items = await getAllItems();
  setGallery(items);
}, []);

// Open modal from gallery card action button
const openModal = (mode: GalleryMode, item: GalleryItem) => {
  if (mode === 'input') return; // can't use input as a mode
  setActiveModal({ mode: mode as any, sourceItem: item });
};

// Save result from any modal back to gallery
const saveResult = async (
  dataUrl: string,
  mode: GalleryMode,
  sourceItem: GalleryItem,
  prompt: string,
  metadata: ModeMetadata
) => {
  const blob = dataUrlToBlob(dataUrl);
  const img = await loadImage(dataUrl);
  const item: GalleryItem = {
    id: crypto.randomUUID(),
    filename: generateFilename(sourceItem.filename, mode),
    dataUrl,
    blob,
    mode,
    prompt,
    parentId: sourceItem.id,
    parentFilename: sourceItem.filename,
    width: img.naturalWidth,
    height: img.naturalHeight,
    createdAt: Date.now(),
    metadata,
  };
  await saveItem(item);
  await refreshGallery();
  setActiveModal(null);
};
```

---

### Rendering Logic

#### Gallery Filtering

```typescript
const filteredGallery = useMemo(() => {
  if (filter === 'all') return gallery;
  return gallery.filter(item => item.mode === filter);
}, [gallery, filter]);
```

#### Gallery Card Render

```tsx
<div className="gallery-card">
  <input type="checkbox" checked={selectedIds.has(item.id)}
    onChange={() => toggleSelect(item.id)} />
  <span className={`mode-badge mode-${item.mode}`}>{item.mode.toUpperCase()}</span>
  <img src={item.dataUrl} onDoubleClick={() => setLightboxItem(item)} />
  <div className="card-meta">
    <span className="filename">{item.filename}</span>
    <span className="dims">{item.width}x{item.height} · {formatBytes(item.blob.size)}</span>
  </div>
  <div className="card-actions">
    <button onClick={() => openModal('inpaint', item)} title="Inpaint">🖌️</button>
    <button onClick={() => openModal('replace', item)} title="Replace">✂️</button>
    <button onClick={() => openModal('edit', item)} title="EDIT">✏️</button>
    <button onClick={() => openModal('outpaint', item)} title="Outpaint">🖼️</button>
  </div>
</div>
```

#### Modal Router

```tsx
{activeModal?.mode === 'inpaint' && (
  <InpaintModal source={activeModal.sourceItem} settings={settings}
    onSave={(dataUrl, prompt, meta) => saveResult(dataUrl, 'inpaint', activeModal.sourceItem, prompt, meta)}
    onClose={() => setActiveModal(null)} />
)}
{activeModal?.mode === 'replace' && (
  <ReplaceModal source={activeModal.sourceItem} settings={settings}
    onSave={(dataUrl, prompt, meta) => saveResult(dataUrl, 'replace', activeModal.sourceItem, prompt, meta)}
    onClose={() => setActiveModal(null)} />
)}
{activeModal?.mode === 'edit' && (
  <EditModal source={activeModal.sourceItem} settings={settings}
    onSave={(dataUrl, prompt, meta) => saveResult(dataUrl, 'edit', activeModal.sourceItem, prompt, meta)}
    onClose={() => setActiveModal(null)} />
)}
{activeModal?.mode === 'outpaint' && (
  <OutpaintModal source={activeModal.sourceItem} settings={settings}
    onSave={(dataUrl, prompt, meta) => saveResult(dataUrl, 'outpaint', activeModal.sourceItem, prompt, meta)}
    onClose={() => setActiveModal(null)} />
)}
```

---

### ZIP Export

```typescript
import JSZip from 'jszip';

async function exportZip(items: GalleryItem[]): Promise<void> {
  if (items.length === 0) { alert('No results to export.'); return; }
  const zip = new JSZip();
  for (const item of items) {
    zip.file(item.filename, item.blob);
  }
  const content = await zip.generateAsync({ type: 'blob' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(content);
  a.download = 'banana-tool-export.zip';
  a.click();
  URL.revokeObjectURL(a.href);
}
```

---

### Style Guide

#### CSS Variables (`index.css`)

```css
:root {
  --bg: #111113;
  --surface: #1c1c1f;
  --surface2: #252528;
  --border: #333338;
  --text: #e8e8ed;
  --text-dim: #8e8e93;
  --accent: #f5c542;
  --accent-hover: #e6b832;
  --danger: #e04848;
  --danger-hover: #c93030;
  --radius: 8px;
  --radius-lg: 12px;

  /* Mode colours */
  --mode-input: #8e8e93;
  --mode-inpaint: #f5c542;
  --mode-replace: #63B3ED;
  --mode-edit: #6bffb8;
  --mode-outpaint: #e09eff;

  /* Status colours */
  --status-idle: #8e8e93;
  --status-processing: #ffe066;
  --status-done: #6bffb8;
  --status-error: #ff6b6b;
}
```

→ Theme adapted from `REF:INPAINT` `src/index.css` with mode colour extensions.

#### Layout

- Full-viewport app, no scrollbar on body. Gallery area scrolls independently.
- Font: `'Segoe UI', system-ui, -apple-system, sans-serif`.
- All elements: `box-sizing: border-box`.
- Gallery grid: `display: grid; grid-template-columns: repeat(auto-fill, minmax(180px, 1fr)); gap: 12px;`.
- Modals: fixed-position overlays with backdrop `rgba(0,0,0,0.75)`, centered content with `max-width: 1100px; max-height: 90vh; overflow-y: auto;`.
- Responsive: modals stack vertically below 768px width.

---

### Tauri Configuration

#### `src-tauri/tauri.conf.json` (key fields)

```json
{
  "productName": "Banana-Tool",
  "version": "1.0.0",
  "identifier": "com.bananatool.app",
  "build": {
    "frontendDist": "../dist"
  },
  "app": {
    "title": "Banana-Tool",
    "windows": [{
      "width": 1280,
      "height": 800,
      "resizable": true,
      "title": "Banana-Tool"
    }]
  },
  "bundle": {
    "active": true,
    "targets": ["msi"],
    "icon": ["icons/32x32.png", "icons/128x128.png", "icons/icon.ico"],
    "windows": {
      "wix": { "language": "en-US" }
    }
  }
}
```

#### `src-tauri/capabilities/default.json`

```json
{
  "identifier": "default",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "dialog:default",
    "fs:default"
  ]
}
```

---

## Edge Cases

- **No API key set**: all Generate buttons disabled; Settings modal shows prominent warning.
- **No image loaded**: gallery shows empty state with drop zone hint. Mode buttons on cards are the only entry point — no cards means no modals.
- **Large images (>4K)**: warn user before processing. Gemini API can fail on oversized inputs. Consider downscaling before sending.
- **Inpaint with no mask painted**: Generate button disabled. Tooltip: "Paint a mask first".
- **Replace with box < 64×64**: Crop button disabled. Tooltip: "Selection too small (min 64×64)".
- **Outpaint with no prompt**: Generate button disabled. Outpaint requires a prompt (unlike other modes which have defaults).
- **API timeout (>120s)**: AbortController fires. Error status: "Request timed out (120s)."
- **API returns text only**: error status: "API returned text only, no image."
- **IndexedDB unavailable**: app still functions but shows warning banner; results cannot be persisted.
- **Modal close during processing**: show confirm dialog "Generation in progress. Cancel?"
- **Filename inheritance chain**: filenames can grow long through chained edits. Truncate base name to 60 chars before appending mode + timestamp.
- **Drag-and-drop multiple files**: each file added as separate `input` gallery item.
- **Concurrent API calls**: only one modal open at a time (guaranteed by single `activeModal` state). No concurrent API calls needed.

---

## Build & Distribution

### Development

```bash
cd banana-tool
npm run dev          # Vite dev server (web)
npm run tauri dev    # Tauri dev (desktop with hot reload)
```

### Production Build

```bash
npm run build              # Vite production build → dist/
npm run tauri build        # Tauri MSI build → src-tauri/target/release/bundle/msi/
```

The MSI installer is portable and self-contained. No runtime dependencies required on the target machine.

---

## Implementation Sequence

### Phase 1 — Scaffold & Core
1. Initialize Vite + React + TypeScript project.
2. Set up Tauri 2.
3. Create `types.ts` with all type definitions.
4. Implement `lib/settings.ts` (localStorage persistence).
5. Implement `lib/galleryDb.ts` (IndexedDB CRUD).
6. Implement `lib/gemini.ts` (API client with all mode wrappers).
7. Implement `lib/imageUtils.ts` (canvas helpers).
8. Implement `lib/filenaming.ts`.
9. Create `index.css` with theme variables.

### Phase 2 — Gallery UI
1. `App.tsx` — root state, gallery loading, filter logic.
2. `Header.tsx` — title bar + settings button.
3. `FilterBar.tsx` — mode filter toggles.
4. `GalleryGrid.tsx` + `GalleryCard.tsx` — responsive grid with cards.
5. `GalleryToolbar.tsx` — load/download/delete/ZIP actions.
6. `Lightbox.tsx` — full-size preview overlay.
7. `SettingsModal.tsx` — settings form.
8. Drag-and-drop image loading.
9. `App.css` — all component styles.

### Phase 3 — Shared Modal Components
1. `ModalShell.tsx` — backdrop, close button, two-column layout.
2. `PromptInput.tsx` — reusable textarea.
3. `ResolutionPicker.tsx` — 1K/2K/4K button group.
4. `Image2Picker.tsx` — optional reference image upload.
5. `StatusLine.tsx` — processing status display.
6. `ResultPreview.tsx` — result image + Save/Regenerate/Discard.

### Phase 4 — EDIT Modal (simplest mode)
1. `EditModal.tsx` — image preview + prompt + strength + resolution + Image 2.
2. Wire up `geminiEdit()` call.
3. Wire up `saveResult()` back to gallery.
4. Test end-to-end.

### Phase 5 — Inpaint Modal
1. Adapt `useMaskEditor.ts` hook from `REF:INPAINT`.
2. `CanvasEditor.tsx` — dual canvas with mouse/touch handling.
3. `BrushToolbar.tsx` — brush controls.
4. `InpaintModal.tsx` — full inpaint UI with single/multi toggle.
5. Wire up `geminiInpaint()` / `geminiInpaintMulti()`.
6. Test end-to-end.

### Phase 6 — Replace Modal
1. Adapt `useCropBox.ts` hook from `REF:REPLACE`.
2. Adapt `lib/colourLab.ts` from `REF:REPLACE`.
3. `CropCanvas.tsx` — crop box canvas with overlay.
4. `ReplaceModal.tsx` — full replace UI with strength, colour match, edge blend.
5. Wire up `geminiReplace()` + post-processing pipeline (upscale → API → colour match → downscale → edge blend → composite).
6. Test end-to-end.

### Phase 7 — Outpaint Modal
1. Adapt `useOutpaintLayout.ts` hook from `REF:OUTPAINT`.
2. `OutpaintModal.tsx` — SVG preview + canvas ratio + scale + prompt.
3. Wire up composite canvas builder + `geminiOutpaint()`.
4. Test end-to-end.

### Phase 8 — Polish & Build
1. Responsive layout testing.
2. Error handling review.
3. ZIP export testing.
4. Tauri build configuration.
5. MSI build and smoke test.
