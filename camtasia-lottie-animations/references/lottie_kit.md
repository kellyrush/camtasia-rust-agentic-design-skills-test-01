# lottie_kit.py

Save the code below as `lottie_kit.py` in your working folder and import it (`from lottie_kit import *`). Builders (`anim`, `trim`, `repeater`, `layer`, null rigs) keep the final keyframe bare and every key below `op`; `lint(doc)` raises on the rules that make a Lottie vanish or be refused (see SKILL.md). Run `lint()` before every `json.dump`. `python3 lottie_kit.py` runs a smoke test.

```python
#!/usr/bin/env python3
"""lottie_kit — battle-tested builders for Camtasia-safe Lottie JSON.

Enforces the two rules that make layers silently vanish in Camtasia when broken:
  1. The FINAL keyframe of every animated property is bare {"t", "s"} (no easing).
  2. All keyframes sit strictly below op (and visible end-states should land
     at or before op-2, the last playable frame).

Usage:
    from lottie_kit import *
    d = doc("My Anim", 180, [ ...layers... ])
    lint(d)                       # raises on rule violations
    json.dump(d, open(path, "w"))

See the design cookbook (references/design-cookbook.md) for the recipes these
builders implement: draw-ons, bursts, ripples, matte shine sweeps, null rigs,
and the drain / whip-around / retract stroke exits.
"""
import json
import math

# ---------------- easing ----------------

def ease(d, ox, oy, ix, iy):
    """Cubic-bezier influence pair for d-dimensional values."""
    return ({"x": [ox] * d, "y": [oy] * d}, {"x": [ix] * d, "y": [iy] * d})

SWISH = lambda d=1: ease(d, 0.7, 0.0, 0.15, 1.0)    # hero moves: fast start, soft settle
SMOOTH = lambda d=1: ease(d, 0.333, 0.0, 0.667, 1.0)  # pulses/breathing: gentle S-curve


def anim(keys):
    """Animated property from [(t, [values], easing_or_None), ...].

    Easing applies to every key EXCEPT the last, which is emitted bare —
    Camtasia drops the whole layer if the final key carries i/o dicts.
    """
    out = []
    for n, (t, s, e) in enumerate(keys):
        if n == len(keys) - 1:
            out.append({"t": t, "s": s})
        else:
            o, i = e if e else SMOOTH(len(s))
            out.append({"t": t, "s": s, "o": o, "i": i})
    return {"a": 1, "k": out}


def st(v):
    """Static property."""
    return {"a": 0, "k": v}


# ------------- trim patterns (see cookbook: entrances & exits) -------------

def draw_e(t0, t1):
    """Trim 'e' draw-on: stroke draws 0 -> 100."""
    return anim([(t0, [0], SWISH()), (t1, [100], None)])


def draw_undraw(t_draw0, t_draw1, t_un0, t_un1):
    """Trim 'e' that draws 0->100, holds, then retracts 100->0 (reverse exit)."""
    return anim([(t_draw0, [0], SWISH()), (t_draw1, [100], SMOOTH()),
                 (t_un0, [100], SWISH()), (t_un1, [0], None)])


def drain_s(u0, u1):
    """Trim 's' 0->100: stroke runs FORWARD off its own tail (drain exit)."""
    return anim([(u0, [0], SWISH()), (u1, [100], None)])


def whip_o(base, u0, u1, end):
    """Trim offset spin (e.g. -90 -> -450) — pair with a collapsing 'e' for the
    whip-around exit on closed outlines (rings, rims, bulb glass)."""
    return anim([(u0, [base], SWISH()), (u1, [end], None)])


# ---------------- shape items ----------------

def tr(p=(0, 0), s=(100, 100), r=0, o=100, p_anim=None, s_anim=None):
    """Group transform. Every gr requires one as its last item."""
    return {"ty": "tr", "p": p_anim or st(list(p)), "a": st([0, 0]),
            "s": s_anim or st(list(s)), "r": st(r), "o": st(o),
            "sk": st(0), "sa": st(0), "nm": "Transform"}


def group(name, items, transform=None):
    return {"ty": "gr", "nm": name, "it": items + [transform or tr()],
            "bm": 0, "hd": False}


def path(verts, in_t=None, out_t=None, closed=False):
    """Bezier path; tangents default to zero (straight lines)."""
    n = len(verts)
    return {"ind": 0, "ty": "sh", "ix": 1, "hd": False, "nm": "Path",
            "ks": {"a": 0, "k": {"i": in_t or [[0, 0]] * n,
                                 "o": out_t or [[0, 0]] * n,
                                 "v": verts, "c": closed}}}


def ellipse(cx, cy, w, h):
    return {"ty": "el", "d": 1, "s": st([w, h]), "p": st([cx, cy]), "nm": "Ellipse"}


def rect(cx, cy, w, h, r=0, p_anim=None):
    """p_anim lets burst ticks FLY — trim alone only shrinks them in place."""
    return {"ty": "rc", "d": 1, "s": st([w, h]), "p": p_anim or st([cx, cy]),
            "r": st(r), "nm": "Rect"}


def stroke(rgb, w, opacity=100, w_anim=None):
    """Round caps/joins (house style). w_anim enables the ripple width->0 pop.

    opacity may be an anim(...) dict, same as fill(). It MUST be passed through rather
    than st()-wrapped: wrapping an anim dict yields {"a":0,"k":{...}} — a static property
    whose k is a dict — which Camtasia's Lottie loader refuses outright with
    "No appropriate stats loader", as an uncatchable host exception.
    """
    op = opacity if isinstance(opacity, dict) else st(opacity)
    return {"ty": "st", "c": st(list(rgb) + [1]), "o": op,
            "w": w_anim or st(w), "lc": 2, "lj": 2, "bm": 0,
            "nm": "Stroke", "hd": False}


def fill(rgb, opacity=100):
    """opacity may be an anim(...) dict for light-on / state pops."""
    op = opacity if isinstance(opacity, dict) else st(opacity)
    return {"ty": "fl", "c": st(list(rgb) + [1]), "o": op, "r": 1,
            "bm": 0, "nm": "Fill", "hd": False}


def trim(s, e, o=0):
    """Trim paths, mode 1 only (mode 2 is unproven in Camtasia — avoid)."""
    mk = lambda v: v if isinstance(v, dict) else st(v)
    return {"ty": "tm", "s": mk(s), "e": mk(e), "o": mk(o), "m": 1,
            "nm": "Trim", "hd": False}


def repeater(copies, rot):
    """Radial repeat: rot = 360/copies for a full circle."""
    return {"ty": "rp", "c": st(copies), "o": st(0), "m": 1, "nm": "Repeater",
            "hd": False,
            "tr": {"ty": "tr", "p": st([0, 0]), "a": st([0, 0]),
                   "s": st([100, 100]), "r": st(rot),
                   "so": st(100), "eo": st(100), "nm": "Transform"}}


# ---------------- layers ----------------

def layer(ind, name, shapes, op, ks_over=None, parent=None, ip=0):
    """Shape layer. Use ip/op to gate one-shot elements (bursts, ripples,
    shine passes). ks_over overrides transform props (p/a/s/r/o)."""
    ks = {"o": st(100), "r": st(0), "p": st([500, 500, 0]),
          "a": st([0, 0, 0]), "s": st([100, 100, 100])}
    if ks_over:
        ks.update(ks_over)
    L = {"ddd": 0, "ind": ind, "ty": 4, "nm": name, "sr": 1, "ks": ks,
         "ao": 0, "shapes": shapes, "ip": ip, "op": op, "st": 0, "bm": 0}
    if parent is not None:
        L["parent"] = parent
    return L


def null_layer(ind, name, op, ks_over=None):
    """Null rig for pop/shake/settle/breathing across children.
    Children set parent=<ind> and usually p=[0,0,0]. Opacity does NOT inherit."""
    ks = {"o": st(0), "r": st(0), "p": st([500, 500, 0]),
          "a": st([0, 0, 0]), "s": st([100, 100, 100])}
    if ks_over:
        ks.update(ks_over)
    return {"ddd": 0, "ind": ind, "ty": 3, "nm": name, "sr": 1, "ks": ks,
            "ao": 0, "ip": 0, "op": op, "st": 0}


def matte_pair(source_layer, consumer_layer):
    """Alpha track matte: source must sit IMMEDIATELY ABOVE consumer in the
    layers array. Returns (source, consumer) tagged; place adjacently."""
    source_layer["td"] = 1
    consumer_layer["tt"] = 1  # 2 = inverted alpha
    return source_layer, consumer_layer


def doc(name, op, layers, fr=30, w=1000, h=1000):
    return {"v": "5.10.2", "fr": fr, "ip": 0, "op": op, "w": w, "h": h,
            "nm": name, "ddd": 0, "assets": [], "layers": layers,
            "markers": []}


# ---------------- lint ----------------

def lint(d):
    """Raise ValueError on the silent-layer-drop rules + the fatal stroke-opacity rule.

    Run before saving. Catches:
      1. easing dict on a final keyframe   -> layer silently dropped
      2. keyframe at/past op               -> layer silently dropped
      3. DOUBLE-WRAPPED PROPERTY, i.e. {"a":0,"k":{...}} -> the whole FILE is refused by
         Camtasia's loader ("IOException: No appropriate stats loader"), as an
         uncatchable host exception in Camtasia Editor. This happens when
         an anim(...) dict is passed somewhere that st()-wraps its argument. A static
         property's k must be a number or a list, NEVER a dict.
    """
    probs = []

    def walk(o, ps):
        if isinstance(o, dict):
            # Double-wrap signature: k is itself a PROPERTY (has both 'a' and 'k').
            # A dict-valued k is otherwise legitimate — a bezier path's ks.k is the
            # {i,o,v,c} path object — so don't flag merely on dict-ness.
            k = o.get("k")
            if o.get("a") == 0 and isinstance(k, dict) and "a" in k and "k" in k:
                probs.append(f"{ps}: double-wrapped property {{'a':0,'k':{{'a':..,'k':..}}}} — "
                             "an anim() dict got st()-wrapped; Camtasia refuses the whole file")
            if o.get("a") == 1 and isinstance(o.get("k"), list) and o["k"] \
                    and isinstance(o["k"][0], dict):
                last = o["k"][-1]
                if "i" in last or "o" in last:
                    probs.append(f"{ps}: easing dict on FINAL keyframe")
                if last.get("t", 0) >= d["op"]:
                    probs.append(f"{ps}: keyframe at/past op ({last.get('t')})")
            for k, v in o.items():
                walk(v, ps + "." + str(k))
        elif isinstance(o, list):
            for n, v in enumerate(o):
                walk(v, ps + f"[{n}]")

    walk(d, d.get("nm", "doc"))
    if probs:
        raise ValueError("Camtasia-unsafe Lottie:\n  " + "\n  ".join(probs))
    return True


# ---------------- geometry helpers ----------------

def regular_polygon(R, sides, start_deg=None):
    """Vertex list for path(closed=True). start_deg default gives a flat-top
    octagon (stop-sign orientation) when sides=8."""
    a0 = (180.0 / sides) if start_deg is None else start_deg
    return [[round(R * math.cos(math.radians(a0 + 360.0 * k / sides)), 2),
             round(R * math.sin(math.radians(a0 + 360.0 * k / sides)), 2)]
            for k in range(sides)]


if __name__ == "__main__":
    # Smoke test: minimal draw-on ring, linted.
    OP = 90
    ring = layer(1, "ring", [group("ring", [
        ellipse(0, 0, 800, 800),
        trim(0, draw_e(0, 24), o=-90),
        stroke([0.141, 0.698, 0.42], 30),
    ])], OP)
    d = doc("smoke", OP, [ring])
    lint(d)
    print(json.dumps(d)[:120] + "... OK")
```
