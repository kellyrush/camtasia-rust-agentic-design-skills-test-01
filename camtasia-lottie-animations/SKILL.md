---
name: camtasia-lottie-animations
description: "For CamRust. Author and use Lottie JSON animations: animated icons, badges, spinners, bursts and seamless loops, plus a retro pixel (8-bit, LCD, dot-matrix) style with an image-to-pixel-art option. Covers the verified feature set, house-style design cookbook, loop and trim rules, Python builders, and a CamRust import-and-render QA loop."
---

# Lottie Animations for CamRust

CamRust imports Lottie JSON as ordinary media: `import_media` the `.json`, then
place it with `add_media_to_timeline`. It rasterizes with ThorVG, the same
engine Camtasia uses, and the file's unique fill/stroke colors become
`Color000`, `Color001`… properties the user can recolor. Files you author here
should also open in Camtasia Editor, so the rules below cover both.

Evidence base: a feature census of 342 known-working files (290 TechSmith-shipped
assets + 52 third-party) plus live render verification of every feature class
below. Camtasia's own library uses trim paths in 67% of its files; the feature
set is much broader than "basic."

## Supported vs. unsupported

**SAFE (corpus-proven and render-verified):**
- Shape layers (`ty:4`) with groups `gr`, paths `sh`, ellipses `el`, rects `rc`
  (incl. rounded `r`), fills `fl`, strokes `st` (width/caps/joins)
- **Trim paths (`tm`) mode 1**: animated `s`, `e` and `o` (offset), including
  negative and spinning offsets (draw-ons, drains, whip-arounds). Mode 2: 0/342 files, avoid.
- **Alpha track mattes** (`tt:1`, `tt:2` inverted; source `td:1` directly above
  the consumer). Luma mattes: 0 files, avoid.
- **Repeaters (`rp`)**: radial bursts, tick dials (verified to 45 copies)
- **Precomps** (assets with `layers`, `refId`); **merge paths (`mm`) mode 1**
- **Gradients (`gf`)** linear and radial
- Layer **parenting**, incl. null-layer (`ty:3`) rigs for pop/shake/settle
- **Animated bezier paths** (path morph); masks (`masksProperties`)
- Layer `ip`/`op` windows to gate one-shot elements (bursts, ripples, shine sweeps)
- Animated group transforms (`tr.p`, `tr.s`), stroke width, fill and stroke
  opacity, primitive position (`rc.p`, `el.p`)
- Fractional keyframe times (`t: 20.5`), multi-layer stagger via `st`/`ip`
- Colors as 0–1 floats; hold keyframes (`h:1`)

**AVOID:**
- **Expressions**: not run; CamRust's skill import rejects a Lottie that has
  them. Bake everything (TechSmith bakes a 120 s dial loop as 61 rotation keys).
- **Text layers, fonts, embedded images**: zero usage; unsupported.
- Luma mattes, trim mode 2, layer effects (`ef`), layer time remap.

## Critical mechanics

0. **The FINAL keyframe of every animated property must be bare**
   (`{"t": N, "s": [...]}`, no `i`/`o` easing dicts), and every key strictly
   below `op`. A trailing easing dict makes Camtasia silently drop the whole
   layer (0/290 TechSmith files violate this). Long baked loops are fine once
   both rules hold.
1. **Never double-wrap a property.** A static property whose `k` is itself a
   property (`{"a":0,"k":{"a":..,"k":..}}`) makes Camtasia refuse the entire
   file ("No appropriate stats loader"). It comes from passing an `anim(...)`
   dict into a builder argument that `st()`-wraps it. If you add a builder,
   mirror `fill()`: `x if isinstance(x, dict) else st(x)`.
2. **Intrinsic duration is a hard ceiling: trim down, never extend.** Use the
   `duration_s` that `import_media` returns (≈ `(op - ip)/fr`); never compute or
   exceed it. Nothing is drawn from that point on. Pass a shorter `duration_s`
   or `media_end_s` to `add_media_to_timeline` to cut before an exit. Land any
   "must end empty" key at or before `op-2` so the last playable frame is clean.
   To fill a long span, author a seamless loop and place it again (or make a
   longer file); don't stretch it.
3. **`layer()` in `lottie_kit` defaults `ks.p` to `[500,500,0]`, correct only for
   a 1000x1000 canvas.** On a smaller canvas every layer sits outside the frame
   and the file renders **blank while every other signal says success** (import
   OK, size and duration right, `lint()` passes). Pin the layer to the origin:
   ```python
   _layer = layer
   def layer(ind, name, shapes, op, ks_over=None, parent=None, ip=0):
       ks = {"p": st([0, 0, 0])}
       if ks_over: ks.update(ks_over)
       return _layer(ind, name, shapes, op, ks, parent, ip)
   ```
4. Scale/position/opacity animation and effects work on Lottie clips like any
   visual media.
5. **Generate programmatically** (Python + `json.dump`); hand-editing keyframe
   JSON invites rule-0 failures. Use `lottie_kit.py` from
   [references/lottie_kit.md](references/lottie_kit.md) (save the code block as
   `lottie_kit.py`): builders (`anim`, `trim`, `repeater`, `layer`, null rigs)
   that keep final keys bare, and `lint(doc)`, which catches rules 0 and 1. Run
   `lint()` before every save.

## Design rules

- **Opacity over dark backgrounds reads muddy**: a pale fill mid-fade over a dark
  canvas looks olive, not glowing. For state changes pop a solid fill 0→100 fast
  ("a light turns on"); for flashes hold near-full opacity ~60% of life, then fade late.
- **Burst ticks must FLY, not just trim**: animate the primitive's position
  outward (`rc.p`) together with the trim, as TechSmith's Bursts do.
- **Accents, not the show**: 1–4 s one-shots; motion completes by ~60% of the
  file and holds. Keep loops slow and subtle.

Full recipes (entrances, holds, the three stroke exits: drain / whip-around /
retract, seamless-loop structure, easing values, staggers, canvas and color
conventions, corpus crib index): **read
[references/design-cookbook.md](references/design-cookbook.md).**

## Style option: RETRO PIXEL (80s LCD / Game Boy / 8-bit)

A deliberately non-house style: monochrome chunky **dot-matrix cell art**, motion
that **snaps** cell-to-cell via hold keyframes, and **Bayer dither** reveals and
dissolves. Trigger words: retro pixel, pixel art, 8-bit / 16-bit, Game Boy, DMG,
LCD, dot matrix, LED sign, old Macintosh, 1-bit, "pixelate this".

Settle two things with the user BEFORE building:
1. **Lifecycle**: one-shot (dither in / out, ends empty) *or* seamless loop. Ask.
2. **Resolution vs. aspiration**: the cell budget decides whether a subject
   reads. Geometric objects and simple glyphs work at 16×16; a specific
   recognisable character often won't. Warn early and preview the bitmap as a
   flat PNG before building.

Tooling in [references/pixel_tools.md](references/pixel_tools.md): `retropix.py`
(Sprite + build, `lcd` and `tint` skins, both lifecycles) and `pixelize.py`
(real image → cell string). Save them next to `lottie_kit.py`. Tokens, animation
grammar, layer stack, resolution table and traps:
**read [references/retro-pixel-style.md](references/retro-pixel-style.md).**

## QA loop in CamRust (required; never assume)

"It imported" is not evidence that it renders. Work in a scratch project
(`project_new`, `project_close` when done) so QA imports don't litter the user's
Media Bin, and QA every edit under a fresh versioned filename (`spin_v2.json`).

1. **Import**: `import_media` the `.json`. An error means the file is refused
   (bad JSON, no `w`/`h`); note the returned `duration_s` and `Color000..` list.
2. **Plate**: for a transparent asset, first put a bright solid (import a
   white or light PNG) on the bottom track. Over a dark canvas dark ink is
   invisible and you can't judge alpha.
3. **Place** the Lottie above it with `add_media_to_timeline`.
4. **Render** with `render_frame` (`inline: true`) at **at least two
   mid-animation times plus the last playable frame** (`duration_s` minus one
   frame) and check:
   - **Colors** right? (wrong or black fills = unsupported feature)
   - **Motion** happening? (compare positions between frames)
   - Element present at all? (missing layer = rule 0 or an unsupported feature)
   - Exits and loops: is the last playable frame empty or at rest? For a loop,
     frame 0 and the last frame should match pixel for pixel.
5. **Count ink numerically** over the bright plate, since a blank render is easy
   to miss:
   ```python
   a = np.array(Image.open(frame).convert("RGB"))
   print("non-white px:", ((a.astype(int).sum(2)) < 720).sum())   # 0 == nothing rendered
   ```
6. Iterate: fix → re-import under a new name (`replace_media` swaps it in) →
   re-render the same times. One fix often creates another problem (moving an
   exit earlier can collide with a hold pulse).

## Recipe

1. Generate the JSON with `lottie_kit.py` (compute geometry; `json.dump`).
2. `lint()` it: final keys bare, all keys < `op`, no double-wrapped property.
3. Import and render in CamRust as above; verify color AND motion AND a clean ending.
4. Trim down with `duration_s` / `media_end_s`; never extend.
5. Add one feature at a time and re-QA each; rename to the final filename last.

Companion skill: `camtasia-dynamic-backgrounds` (shader backgrounds; same
fresh-filename and render-QA habits).
