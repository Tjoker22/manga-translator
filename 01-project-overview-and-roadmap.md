# Manga Translator — Project Overview & Roadmap

## What We're Building

A local, open-source application that can read manga and doujin pages (Japanese, Chinese, or Korean), extract the text, translate it into English, erase the original text without damaging the artwork, and render the English translation back into the original speech bubbles and text regions.

Reference products for inspiration:
- Fakey – Manga Translator: https://pawakalabs.com/products/fakey/
- Torii Image Translator: https://toriitranslate.com/

---

## Core Pipeline

Every version of this app — from MVP to full product — runs the same five-stage pipeline:

```
Input Image → Detect → OCR → Translate → Inpaint → Typeset → Output Image
```

| Stage | What it does |
|---|---|
| Detect | Find all text regions (speech bubbles, captions, sound effects) on the page |
| OCR | Read the raw Japanese/Chinese/Korean characters from each region |
| Translate | Convert the extracted text to English |
| Inpaint | Erase the original text pixels without damaging the art underneath |
| Typeset | Render the English translation back into the cleaned region |

---

## Chosen Tool Stack

All local, all open-source, zero ongoing cost after setup.

| Stage | Tool | Why |
|---|---|---|
| Detection + OCR | `manga-ocr` | Vision transformer fine-tuned specifically on manga; handles vertical text, handwriting styles, and sound effects; returns bounding boxes and text together |
| Inpainting | `LaMa` via `lama-cleaner` | State-of-the-art large-mask inpainting; was trained for exactly this use case; runs fully local |
| Translation | `Qwen2.5 7B` via `Ollama` | Strong Japanese→English quality; fits in 6 GB VRAM; no content restrictions; free to run |
| Image manipulation | `Pillow` + `OpenCV` | Cropping regions, drawing text, compositing layers, saving output |
| Typesetting fonts | `Noto Sans CJK` | Full JP/CN/KR glyph support; free from Google; condensed variants fit bubbles well |

### Translation backend options (configurable)

```
TRANSLATION_BACKEND = "ollama"   # default — free, local, no content restrictions
TRANSLATION_BACKEND = "gemini"   # free tier, 1500 req/day, SFW content only
TRANSLATION_BACKEND = "claude"   # best quality, paid API, SFW content only
```

### Why local-first matters

- Works completely offline
- No API costs
- No content policy restrictions — handles adult doujin content
- No data sent to external servers
- GPU acceleration available for all three heavy stages

---

## Development Phases

### Phase 1 — MVP CLI (target: 2–4 weeks)

A Python script you run from the terminal. Point it at a folder of manga page images, it processes them and saves translated images to an output folder.

Goals:
- End-to-end pipeline working on a single page
- Each stage is a separate Python module (easy to test and swap out)
- Intermediate images saved at each step so you can inspect what's happening
- Config file to switch translation backend

### Phase 2 — Local App / Web Server (target: 4–8 weeks)

Add a drag-and-drop interface so you or others can use it without touching the terminal.

Two sub-routes:
- **2A — FastAPI backend + simple web frontend** (recommended — reuses for Phase 3)
- **2B — Gradio** (10 lines of Python to wrap the pipeline in a web UI; fastest path to a working interface)

Additional goals at this phase:
- Page-level translation context (pass previous page's translations to keep character names and dialogue consistent)
- Manual correction mode for low-quality scans

### Phase 3 — Live Web / Browser Extension (target: 8–16 weeks)

Two sub-routes:
- **3A — Browser extension** (intercepts manga pages in the browser, overlays translated text in-place; best UX, hardest technically)
- **3B — Hosted web app** (users upload chapters, server processes and returns translated images)

---

## Key Technical Challenges

**Vertical text:** Japanese manga often runs top-to-bottom, right-to-left. `manga-ocr` reads it correctly but you need to handle bounding box orientation when placing English text.

**Font fitting:** English text is wider per character than Japanese. Build an auto-size function that shrinks font size and wraps lines until translation fits the bubble.

**Sound effects (SFX):** Large stylized kanji used for sounds are part of the artwork. Use OCR confidence thresholds — skip or flag regions with low confidence rather than inpainting over art.

**Doujin scan quality:** Fan-made doujin has messier, lower-resolution scans than commercial manga. OCR accuracy drops. Manual correction mode in Phase 2 addresses this.

---

## API Key Situation

**Important distinction:** Claude.ai Pro subscription and the Anthropic API are completely separate products. Your Pro plan gives access to the chat interface only — it does not include API credits. The API is billed per token via `console.anthropic.com`.

**Free path (recommended to start):**
1. `Qwen2.5 7B` via Ollama — completely free, runs locally, no account needed
2. Gemini API free tier — 1,500 requests/day, requires Google account, SFW only

Only bring in paid Claude API when comparing quality and ready to budget for it — likely not until Phase 2 or later.

---

## Hardware

**Proxmox Host:**
- CPU: Ryzen 7 5700X (8-core, 16-thread, 3.4 GHz) — no integrated graphics
- GPU: NVIDIA MSI GeForce RTX 2060 (6 GB VRAM)
- RAM: 64 GB installed, 32 GB allocatable to VM

**GPU VRAM fit:**

| Model | VRAM needed | Fits on 2060? |
|---|---|---|
| `manga-ocr` | ~1.5 GB | Yes |
| `LaMa` inpainting | ~2 GB | Yes |
| `Qwen2.5 7B` Q4 | ~4.5 GB | Yes |
| `Qwen2.5 14B` Q4 | ~8.5 GB | No — exceeds VRAM, falls back to CPU RAM (slower but works) |
| `Llama 3.1 8B` Q4 | ~5 GB | Tight but possible |

Start with 7B for speed. Try 14B later knowing it will be slower due to CPU RAM offloading.

**VM allocation:**
- CPU: 8 cores
- RAM: 32 GB
- Disk: 80 GB
- GPU: RTX 2060 via PCIe passthrough
