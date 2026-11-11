# Retro Pixel — 80s LCD / Game Boy style Lottie

A monochrome, chunky dot-matrix look: shapes are drawn as **lit cells on a coarse
square lattice**, motion **snaps** cell-to-cell instead of tweening, and reveals /
exits run in **Bayer dither order**.

Use when the ask sounds like: retro pixel, pixel art, 8-bit / 16-bit, Game Boy,
DMG, LCD, dot matrix, LED sign, old Macintosh, System 6, 1-bit, "make it look
like an old handheld", "pixelate this".

Evidence base: measured from a 22 s classic-Mac pixel screensaver recording
(1318×742 @30fps) plus the TechSmith `Phosphor-Drift` / `Phosphor-Stipple`
shaders, and render-verified in Camtasia 2026.1.0.

---

## 1. Design tokens

| Property | Value | Source |
|---|---|---|
| Dot size | **0.78–0.85 × cell pitch** | measured 0.80; Phosphor-Stipple default 0.78, Drift 0.88 |
| Gap | 15–22 % of pitch | cells must NEVER touch — this is the whole look |
| Corner radius | **0** | hard squares |
| Tones | **1-bit** | greys come from dot DENSITY (Bayer), not grey dots |
| Ghost cells | unlit cells at **5–7 % ink** | Phosphor: `ink = max(0.05, on)`; reads as LCD substrate |
| Drop shadow | **0.2 cell down-right, ink @ 16 %** | measured 8–13 % darkening beside ink |
| Cell pitch | whatever the grid implies | source used 12.9 px at 1318 wide |

**Game Boy DMG palette** — Screen `#7E8E7B` · Ink `#2C3831` · Backlight `#B4CBA6`
· light ink for dark backgrounds `#CDDAC9`.

Author in greyscale/duotone. Colour is best applied later with Camtasia visual
effects, so keep ink to a single flat fill.

### Two skins — build both unless told otherwise
- **`lcd`** — dark ink on an opaque screen panel + ghost grid + drop shadow.
  Self-contained, faithful, needs no backdrop.
- **`tint`** — one flat light ink, transparent, no panel/ghost/shadow.
  For overlay use and Camtasia colour tinting.

⚠️ **Never ship dark ink on transparent** — it is near-invisible over dark
backgrounds. That was the single biggest defect in the first prototype.

---

## 2. Animation grammar

1. **Cells snap, never tween.** Use Lottie **hold keyframes (`h:1`)** on a
   "sprite clock" of ~8–12 fps, and move by a whole number of cells. Sub-pixel
   sliding instantly destroys the retro read. *(Verified working in Camtasia;
   present in 6/345 corpus files including rlottie-verified ones.)*
2. **Cells light up, never fade in.** 0 → 100 % opacity in ONE frame.
3. **Sprite-frame swaps beat transforms.** For a blink, a tail flick, or clock
   hands, define alternate cell sets and toggle which one is visible — the same
   way a real sprite sheet works.

### Lifecycle — ALWAYS ASK THE USER which one
Neither is a default; they suit different jobs.

- **A. One-shot (dither in / dither out).** Bayer reveal ~0.6 s → hold → Bayer
  dissolve ~0.9 s, ending **fully empty at or before `op-2`**. This is the source
  recording's own outro. Best for accents that appear, land a point, and leave.
- **B. Seamless loop.** Always present; only the in-place animation cycles, with
  frame 0 and the last playable frame **pixel-identical**. Best for ambient art
  that sits under narration for an arbitrary length.

The dither dissolve also works as a *transition between two sprites* — dissolve A
out while B dithers in on the same grid.

---

## 3. Construction

Layer stack (top → bottom), from `retropix.py` (in `pixel_tools.md`):

```
part layers      one per animated part (eyes, hands, shutter) so each can swap
base             the static sprite cells
shadow-<part>    offset copy per animated part      ] lcd
shadow-base      offset copy of the base            ] skin
ghost-grid       every cell in the art extent @ 7%  ] only
screen           solid panel                        ]
```

**The efficiency trick:** one group holds many `rc` cells sharing **ONE `fl`**.
Split cells by **Bayer 4×4 class** so a full dither dissolve costs **16 animated
opacity properties, not one per cell.**

```python
from retropix import Sprite, build
sp = Sprite(pad(ART), parts={"eyesopen": [...], "eyesshut": [...]})
build(sp, "retropix-happymac-loop-lcd", skin="lcd", lifecycle="loop",
      op=92, part_schedule={...}, outdir=OUT)
```

- **Pad every sprite by 1 cell.** Unpadded art puts feet and snouts flush against
  the canvas edge and clips the drop shadow.
- **Let the canvas follow the art aspect.** Icons are often wide; forcing a square
  grid wastes half the resolution. `Sprite` computes `cw`/`ch` for you.

---

## 4. Turning a real image into pixel art

The common real request is *"turn this photo/logo into pixels"*, not "hand-draw a
sprite". Use `pixelize.py` (in `pixel_tools.md`):

```python
from pixelize import pixelize, preview
art = pixelize(Image.open("logo.png"), cells_w=24, mode="ink", threshold=0.5)
preview(art, path="check.png")     # ALWAYS eyeball before building the Lottie
```

- `mode="ink"` — 1-bit; a cell lights if dark/opaque enough. Best for line art,
  logos, silhouettes.
- `mode="bayer"` — ordered dither; greys become dot density. Best for photos.
- Transparent pixels always read as paper, never ink.
- `gamma` pushes midtones; `threshold` trades weight for detail.

⚠️ If the SOURCE is already a dot-matrix image, close the inter-cell gaps
(`binary_closing`) before re-quantizing, or the sampling grid straddles gaps and
shreds the shape. Size the kernel to the **gap** (~2–3 px), not the cell — an
11×11 kernel filled an outline's interior into a solid blob.

---

## 5. ⚠️ Resolution vs. aspiration — warn the user early

**Cell budget, not artistic skill, decides whether a subject reads.** Set
expectations BEFORE drawing, and show a preview early.

| Subject type | Minimum that reads |
|---|---|
| Geometric objects (floppy, trash can, watch, document, clover) | **16×16** |
| Symbols / glyphs / simple faces (happy Mac, arrow, hand) | **16×16** |
| Detailed logos, lettering | 24×24+ |
| Organic creatures with a specific identity | **32×32+, often more** |

Real case: **Clarus the dogcow failed at 16×16, 24×24 AND 32×32** across seven
attempts. At 16×16 it was an unrecognisable blob; larger versions read as a
llama, a horse, then an anteater. A *generic quadruped* is achievable; **that
specific character is not**, because its identity lives in fine features (floppy
ear, blunt snout, spots) that need more cells than the silhouette itself.

What this means in practice:
- If a subject is a **specific recognisable character**, say up front that it will
  likely read as a generic version of its category, and offer either a much
  higher cell count or a simpler subject.
- **Preview the bitmap as a flat PNG before building any Lottie** — it is
  seconds, versus a full generate-load-render cycle.
- **Judge at real overlay size (~100 px), not zoomed in.** Plenty of sprites read
  at 500 px and turn to mush at 100.
- Two clean, unmistakable icons beat three where one is a blob. Say so and swap
  the subject rather than shipping the blob.

**Draw in the source's idiom: OUTLINE with occasional solid fills, not solid
silhouettes.** The screensaver's own icons are outline-drawn; solid silhouettes
at low resolution read as featureless blobs. A solid patch in the middle of a
body reads as a wound, not a marking.

---

## 6. QA — verified findings and traps

| Finding | Status |
|---|---|
| **621 `rc` items in one file render fine** | ✅ the largest corpus file has only 24 — this was unproven |
| **Hold keyframes (`h:1`) honoured** — clean one-cell snap, no tween | ✅ |
| Bayer-class grouped dither reveal + dissolve | ✅ |
| Sprite-frame swap via layer-opacity hold keys | ✅ |
| Seamless loops wrap at **0 differing pixels** (frame 0 vs `op-2`) | ✅ |

**Traps that cost real time here:**

- **A shadow layer must be split per animated part.** One shadow layer covering
  all cells left visible ghost blocks where the eyes blinked off.
- **Judge a `tint` asset over a bright plate.** A transparent asset rendered over
  the dark canvas hides dark ink. Put a bright solid on a lower track before
  `render_frame`.
- **QA edits under a fresh filename** (`sprite_v2.json`) so you never compare
  against a stale import.
- **Filling a whole aperture kills the read.** A shutter covering the entire
  floppy window made it look like a calculator grid; a narrow shutter sliding
  inside a still-visible window reads correctly.

Numeric loop check — cheaper and stricter than eyeballing:
```python
a = np.asarray(Image.open("f000.png").convert("RGB"), int)
b = np.asarray(Image.open("f089.png").convert("RGB"), int)
assert (np.abs(a-b).sum(2) > 0).sum() == 0    # frame 0 == last playable
```
