# Banana-Tool

A unified AI-powered image editing desktop app that combines four creative modes — **Inpaint**, **Replace**, **EDIT**, and **Outpaint** — into a single gallery-driven interface. Powered by Google Gemini image generation. 

**UNPAINT** now added, newly created images can use this retrospectively from the gallery, or enable it with new EDIT/REPLACE/INPAINT tasks. This is based on the "DJZ Brushed Layer" post-edit interface and preserves unchanged pixels solving degradation of the image over many edits.

## Download

Get the latest Windows installer from the [Releases](https://github.com/MushroomFleet/banana-tool/releases) page. Download the `.msi` file, run it, and you're ready to go — no dependencies required.

## Features

### Four Editing Modes

**Inpaint** — Paint a mask directly onto your image and let AI fill the masked region. Supports two sub-modes:
- *Single*: removes or replaces the masked area based on your prompt.
- *Multi*: blends content from a second reference image into the masked area.

**Replace** — Draw a crop box around any region of your image. The selected area is sent to AI for replacement based on your prompt. Includes automatic colour matching (RGB or perceptual LAB space) and edge blending for seamless compositing back into the original.

**EDIT** — The simplest mode. Send your entire image with a text prompt and receive an AI-edited version. Optionally include a second reference image to guide the edit.

**Outpaint** — Extend your image beyond its borders. Choose a target canvas ratio (1:1, 4:3, 16:9, 3:4, 9:16, 3:2, 2:3), position your source image within the canvas using the interactive preview, and describe the surrounding scene. AI fills in everything around your placed image.

### Unified Gallery

All images — uploads and results from every mode — live in one central gallery. Each result is tagged by its mode, and a filter bar lets you view all items or narrow down to a specific mode. Gallery cards show a thumbnail, filename, dimensions, and file size alongside four action buttons that let you immediately launch any mode using that image as input.

### Controls at a Glance

| Mode | Key Controls |
|------|-------------|
| Inpaint | Brush size, hardness, overlay opacity, draw/erase toggle, clear/invert mask, single/multi mode, optional reference image |
| Replace | Crop box selection (drag to draw, drag to reposition), strength slider, colour match (RGB/LAB), edge blend radius, optional reference image |
| EDIT | Strength slider, optional reference image |
| Outpaint | Canvas ratio selector, scale slider, interactive drag-to-position preview |

All modes share a prompt input, resolution selector (1K / 2K / 4K), and a result preview with Save / Regenerate / Discard actions.

### Gallery Management

- **Load images** via file picker or drag-and-drop (PNG, JPEG, WebP).
- **Select multiple items** with checkboxes for batch download or deletion.
- **ZIP export** — download all gallery items (or the current filtered set) as a single archive.
- **Lightbox** — double-click any thumbnail for a full-size preview with keyboard navigation (arrow keys, Escape to close).
- **Chained editing** — every result goes back into the gallery, so you can run it through another mode immediately. Filenames carry the editing history (e.g. `photo_inpaint_1710700800000_edit_1710700900000.png`).

## Getting Started

1. Launch Banana-Tool.
2. Open **Settings** (top-right) and enter your Google Gemini API key.
3. Choose your preferred model:
   - **Nano Banana 2** (gemini-2.0-flash) — faster, lower cost.
   - **Nano Banana Pro** (gemini-2.0-pro) — higher quality.
4. Load an image into the gallery.
5. Click any mode button on a gallery card to start editing.

## Settings

| Setting | Description |
|---------|-------------|
| API Key | Your Google Gemini API key (stored locally, never transmitted except to the Gemini API) |
| Model | Choose between Nano Banana 2 (fast) and Nano Banana Pro (quality) |
| Default Resolution | Initial resolution for all modals (1K, 2K, or 4K) |
| Output Format | PNG or JPEG for manual downloads and ZIP exports |
| JPEG Quality | 10–100% (shown only when JPEG is selected) |

All settings are saved locally and persist between sessions.

## Tips

- **Inpaint mask**: use a soft brush (low hardness) for natural blending edges, or a hard brush for precise selections. The red overlay shows exactly what's masked.
- **Replace colour matching**: enable LAB mode for perceptually accurate colour transfers, especially when the replacement patch has different lighting than the surroundings.
- **Replace edge blend**: increase the edge blend radius (up to 32px) for smoother transitions between the replaced region and the original image.
- **Outpaint positioning**: scale your source image down (try 40–60%) to give AI more room to generate surrounding content. Drag the blue box to control exactly where your image sits in the final canvas.
- **Resolution trade-offs**: 1K is fastest and works well for iteration. Use 2K or 4K for final output quality.
- **Chained workflows**: outpaint an image to extend its canvas, then inpaint specific regions, then run an EDIT pass for global style adjustments — all without leaving the app.

## Requirements

- Windows 10/11 (x64)
- A valid [Google Gemini API key](https://aistudio.google.com/apikey)
- Internet connection (for API calls)

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{banana_tool,
  title = {Banana-Tool: Unified AI Image Editing with Inpaint, Replace, Edit and Outpaint},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/banana-tool},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
