# Retro pixel tools: pixelize.py and retropix.py

For the RETRO PIXEL style (see `retro-pixel-style.md`). Save each block below under its file name, in the same folder as `lottie_kit.py` (from `lottie_kit.md`); `retropix.py` imports `lottie_kit` from its own folder. Needs Python 3 with numpy and Pillow.

- `pixelize.py`: a real image (photo, logo) to a pixel-art cell string. Always `preview()` the bitmap as a flat PNG before building.
- `retropix.py`: Sprite + build, both skins (`lcd`, `tint`), both lifecycles (`oneshot`, `loop`); writes a linted Lottie `.json`.

## pixelize.py

```python
#!/usr/bin/env python3
"""pixelize — turn a real image into retro-pixel cell art.

The common real-world request is "make this photo/logo/icon into pixel art",
not "hand-draw me a sprite". This does the quantization and hands back an art
string that retropix.Sprite consumes directly.

Modes
  "ink"    1-bit: a cell is lit if it is dark / opaque enough (line art, logos)
  "bayer"  ordered-dither: greys become dot DENSITY, the authentic LCD way
"""
import numpy as np
from PIL import Image

BAYER4 = np.array([[0, 8, 2, 10], [12, 4, 14, 6],
                   [3, 11, 1, 9], [15, 7, 13, 5]], dtype=float)


def _luma(img):
    if img.mode in ("RGBA", "LA"):
        rgba = np.asarray(img.convert("RGBA"), dtype=float) / 255.0
        alpha = rgba[..., 3]
        lum = rgba[..., :3] @ [0.299, 0.587, 0.114]
        # transparent pixels read as "paper", never as ink
        return lum, alpha
    lum = np.asarray(img.convert("L"), dtype=float) / 255.0
    return lum, np.ones_like(lum)


def pixelize(img, cells_w, cells_h=None, mode="ink", threshold=0.5,
             invert=False, autocrop=True, rotate=0.0, gamma=1.0):
    """Return an art string ('#'/'.') of shape cells_h x cells_w."""
    if rotate:
        img = img.rotate(rotate, resample=Image.BICUBIC, expand=True,
                         fillcolor=(255, 255, 255, 0) if img.mode == "RGBA" else 255)

    lum, alpha = _luma(img)
    ink = (1.0 - lum) * alpha          # 1 = solid ink, 0 = paper
    if invert:
        ink = lum * alpha

    if autocrop:
        sig = ink > max(0.25, ink.max() * 0.35)
        if sig.any():
            ys, xs = np.where(sig)
            pad = 1
            y0, y1 = max(0, ys.min()-pad), min(ink.shape[0], ys.max()+1+pad)
            x0, x1 = max(0, xs.min()-pad), min(ink.shape[1], xs.max()+1+pad)
            ink = ink[y0:y1, x0:x1]

    h, w = ink.shape
    if cells_h is None:
        cells_h = max(1, int(round(cells_w * h / w)))

    # area-average down to the cell grid (box filter = the right downsample here)
    acc = np.zeros((cells_h, cells_w))
    ys = (np.arange(cells_h + 1) * h / cells_h).astype(int)
    xs = (np.arange(cells_w + 1) * w / cells_w).astype(int)
    for j in range(cells_h):
        for i in range(cells_w):
            blk = ink[ys[j]:max(ys[j]+1, ys[j+1]), xs[i]:max(xs[i]+1, xs[i+1])]
            acc[j, i] = blk.mean() if blk.size else 0.0

    if gamma != 1.0:
        acc = np.clip(acc, 0, 1) ** gamma

    if mode == "bayer":
        thr = (BAYER4[np.arange(cells_h)[:, None] % 4,
                      np.arange(cells_w)[None, :] % 4] + 0.5) / 16.0
        on = acc > thr
    else:
        on = acc > threshold

    return "\n".join("".join("#" if on[j, i] else "." for i in range(cells_w))
                     for j in range(cells_h))


def preview(art, size=440, dot=0.80, path=None):
    from PIL import ImageDraw
    rows = art.strip("\n").split("\n")
    W = max(len(r) for r in rows); H = len(rows); g = max(W, H)
    pitch = size / g; d_ = pitch * dot
    im = Image.new("RGB", (round(W*pitch), round(H*pitch)), (126, 142, 123))
    dr = ImageDraw.Draw(im)
    for y in range(H):
        for x in range(W):
            cx, cy = (x+0.5)*pitch, (y+0.5)*pitch
            dr.rectangle([cx-d_/2, cy-d_/2, cx+d_/2, cy+d_/2], fill=(118, 133, 115))
    for y, r in enumerate(rows):
        for x, c in enumerate(r):
            if c == "#":
                cx, cy = (x+0.5)*pitch, (y+0.5)*pitch
                dr.rectangle([cx-d_/2, cy-d_/2, cx+d_/2, cy+d_/2], fill=(44, 56, 49))
    if path:
        im.save(path)
    return im
```

## retropix.py

```python
#!/usr/bin/env python3
"""retropix — build 80s-LCD / Game Boy pixel-art Lottie files for Camtasia.

Style spec: RETRO-PIXEL-STYLE.md (measured from classic-mac-pixel-screensaver-01.trec
plus the TechSmith Phosphor shaders).

Two skins   : "lcd"  opaque Game Boy panel + ghost grid + drop shadow
              "tint" flat single-colour ink on transparent, for Camtasia tinting
Two lifecycles: "oneshot" Bayer dither in -> hold -> Bayer dither out (ends empty)
                "loop"    always present, in-place cycle, frame 0 == last frame
"""
import json, os, sys

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from lottie_kit import st, anim, group, rect, fill, layer, doc, lint

CANVAS = 512
FPS = 30

INK_DARK  = [0.173, 0.220, 0.192]   # #2C3831
SCREEN    = [0.494, 0.557, 0.482]   # #7E8E7B
INK_LIGHT = [0.804, 0.855, 0.788]   # #CDDAC9

DOT_FRAC   = 0.80    # measured 0.78-0.85 of cell pitch
GHOST_O    = 7       # unlit-cell ghost ink %
SHADOW_O   = 16      # drop-shadow ink %
SHADOW_OFF = 0.22    # shadow offset, in cells, down-right

BAYER = [[0, 8, 2, 10], [12, 4, 14, 6], [3, 11, 1, 9], [15, 7, 13, 5]]


class Sprite:
    """A cell-grid sprite. `art` is the static base; `parts` are named cell sets
    that can be swapped/blinked independently (drawn ON TOP of the base)."""

    def __init__(self, art, parts=None):
        rows = art.strip("\n").split("\n")
        self.w = max(len(r) for r in rows)
        self.h = len(rows)
        # Cells stay SQUARE: pitch is set by the longer axis, and the shorter
        # axis is centred. Source icons are often wide (the dogcow is ~26x16),
        # so forcing a square grid would waste half the resolution.
        self.grid = max(self.w, self.h)
        # Canvas follows the ART aspect so the LCD panel and ghost grid fit the
        # sprite exactly instead of sitting in a square letterbox.
        self.cw = round(self.w * (CANVAS / self.grid))
        self.ch = round(self.h * (CANVAS / self.grid))
        self.base = [(x, y) for y, row in enumerate(rows)
                     for x, ch in enumerate(row) if ch == "#"]
        self.parts = parts or {}

    @property
    def pitch(self):
        return CANVAS / self.grid

    @property
    def dot(self):
        return self.pitch * DOT_FRAC

    def xy(self, gx, gy, ox=0.0, oy=0.0):
        return (round((gx + 0.5) * self.pitch - self.cw / 2.0 + ox, 2),
                round((gy + 0.5) * self.pitch - self.ch / 2.0 + oy, 2))


def hold(keys):
    """Stepped keyframes — value snaps, never tweens. Final key stays bare."""
    out = []
    for n, (t, s) in enumerate(keys):
        k = {"t": t, "s": s}
        if n != len(keys) - 1:
            k["h"] = 1
        out.append(k)
    return {"a": 1, "k": out}


def snap(t_on, t_off=None):
    """Cell lights up in ONE frame (never fades in); optionally snaps off."""
    ks = [(t_on, [0], None), (t_on + 1, [100], None)]
    if t_off is not None:
        ks += [(t_off, [100], None), (t_off + 1, [0], None)]
    return anim(ks)


def cell_group(sp, name, cells, colour, opacity, ox=0.0, oy=0.0):
    """N rects sharing ONE fill — the trick that keeps big sprites cheap."""
    items = [rect(*sp.xy(gx, gy, ox, oy), sp.dot, sp.dot) for gx, gy in cells]
    return group(name, items + [fill(colour, opacity)])


def bayer_groups(sp, cells, colour, tag, t_in, t_out, reveal, dissolve, ox=0, oy=0):
    """Split cells into 16 Bayer classes so a dither reveal/dissolve costs
    16 animated properties instead of one per cell."""
    out = []
    for b in range(16):
        cs = [c for c in cells if BAYER[c[1] % 4][c[0] % 4] == b]
        if not cs:
            continue
        on = t_in + round(b * reveal / 16)
        off = None if t_out is None else t_out + round(b * dissolve / 16)
        out.append(cell_group(sp, f"{tag}{b:02d}", cs, colour, snap(on, off), ox, oy))
    return out


def build(sprite, name, skin="lcd", lifecycle="oneshot", op=92,
          part_schedule=None, outdir="."):
    """part_schedule: {part_name: [(t, visible_bool), ...]} on the sprite clock."""
    sp = sprite
    ink = INK_DARK if skin == "lcd" else INK_LIGHT
    part_schedule = part_schedule or {}

    if lifecycle == "oneshot":
        REVEAL, DISSOLVE = 18, 26
        t_in, t_out = 0, op - 4 - DISSOLVE
    else:                                   # loop: present the whole time
        REVEAL, DISSOLVE = 0, 0
        t_in, t_out = None, None

    def sprite_opacity(cells, tag, ox=0, oy=0, colour=None):
        colour = colour or ink
        if lifecycle == "oneshot":
            return bayer_groups(sp, cells, colour, tag, t_in, t_out,
                                REVEAL, DISSOLVE, ox, oy)
        return [cell_group(sp, tag, cells, colour, 100, ox, oy)]

    layers, ind = [], 1

    # ---- animated parts on top (each its own layer so it can blink/swap)
    for pname, cells in sp.parts.items():
        sched = part_schedule.get(pname)
        ks = {"p": st([sp.cw / 2, sp.ch / 2, 0])}
        if sched:
            ks["o"] = hold([(t, [100 if v else 0]) for t, v in sched])
        layers.append(layer(ind, f"part-{pname}",
                            sprite_opacity(cells, f"p_{pname}_"), op,
                            ks_over=ks))
        ind += 1

    # ---- static base
    layers.append(layer(ind, "base", sprite_opacity(sp.base, "b"), op,
                        ks_over={"p": st([sp.cw / 2, sp.ch / 2, 0])}))
    ind += 1

    if skin == "lcd":
        ox = oy = sp.pitch * SHADOW_OFF
        # Shadow must be split per animated part, or a blinking part leaves a
        # visible ghost block where its shadow keeps rendering. (v2 QA bug.)
        for pname, cells in sp.parts.items():
            sched = part_schedule.get(pname)
            ks = {"p": st([sp.cw / 2, sp.ch / 2, 0]), "o": st(SHADOW_O)}
            if sched:
                ks["o"] = hold([(t, [SHADOW_O if v else 0]) for t, v in sched])
            layers.append(layer(ind, f"shadow-{pname}",
                                sprite_opacity(cells, f"s_{pname}_", ox, oy,
                                               INK_DARK), op, ks_over=ks))
            ind += 1
        layers.append(layer(ind, "shadow-base",
                            sprite_opacity(sp.base, "sb", ox, oy, INK_DARK), op,
                            ks_over={"p": st([sp.cw / 2, sp.ch / 2, 0]),
                                     "o": st(SHADOW_O)}))
        ind += 1

        all_cells = [(x, y) for y in range(sp.h) for x in range(sp.w)]
        layers.append(layer(ind, "ghost-grid",
                            [cell_group(sp, "ghost", all_cells, INK_DARK, GHOST_O)],
                            op, ks_over={"p": st([sp.cw / 2, sp.ch / 2, 0])}))
        ind += 1
        layers.append(layer(ind, "screen",
                            [group("panel", [rect(0, 0, sp.cw, sp.ch),
                                             fill(SCREEN, 100)])],
                            op, ks_over={"p": st([sp.cw / 2, sp.ch / 2, 0])}))
        ind += 1

    d = doc(name, op, layers, fr=FPS, w=sp.cw, h=sp.ch)
    lint(d)
    p = os.path.join(outdir, name + ".json")
    json.dump(d, open(p, "w"))
    rects = json.dumps(d).count('"ty": "rc"')
    print(f"  {name}.json  {os.path.getsize(p)/1024:6.1f} KB  rects={rects}")
    return p
```
