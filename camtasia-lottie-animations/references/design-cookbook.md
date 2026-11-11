# Camtasia Lottie Design Cookbook

TechSmith house-style patterns, measured from Camtasia's own 290 shipped Lottie assets
and verified by live renders. Use these so generated animations look native to Camtasia
(and LottieFiles-grade), not generic. All numbers are frames at `fr: 30` unless noted.

## Global conventions (measured from the corpus)

| Convention | Value |
|---|---|
| Frame rate | **30** (279/290 files; a few 60) |
| Canvas | **1000×1000** for icons/badges/cursors; content-fit for wide assets (450×32 slider, 800×100 highlight, 180×180 touch) |
| Duration | 1–4 s one-shots (median 2.0 s); loopers authored long (30 s, up to 120 s) and trimmed in-app |
| Version string | `"v": "5.10.2"` |
| Colors | 0–1 floats + alpha: `[1, 0.902, 0.125, 1]`; flat solid fills dominate |
| Stroke style | round caps + joins (`lc:2, lj:2`) for anything hand-drawn/draw-on |
| Loop split | ~half seamless loops, ~half one-shots with end hold |

### Easing vocabulary

| Name | `o` (out) | `i` (in) | Use |
|---|---|---|---|
| AE default | `x:0.167` | `x:0.833` | filler moves (11.6k uses in corpus) |
| SMOOTH | `x:0.333, y:0` | `x:0.667, y:1` | gentle S-curve: pulses, breathing |
| SWISH | `x:0.7–1.0, y:0` | `x:0–0.15, y:1` | hero moves: fast start, soft settle (TechSmith's checkmark uses `o:{x:1,y:0} → i:{x:0,y:1}`) |

Multi-dim properties (scale, 2D position) need `x`/`y` arrays matching the dims.

### Timing / stagger

- Element draw-ons: 10–24 frames each.
- Stagger siblings by **4–6 frames** (letters) or **20 frames** (loader dots, via layer `st`/`ip` offsets — reuse one design, offset the layer, done).
- Press/click: scale `100 → 90 → 100` over ~12 frames (Arrow Click Mac: keys at 15/21/27).
- Attention shake: rotation `0 → -5 → 5 → -2 → 0`, 4 frames apart.
- Pop-in: scale `0 → 110..112 → 100`, keys at 0/10/16, SWISH out then SMOOTH settle.
- Anticipation pop-out: `100 → 106 → 0`, ~6 then ~10 frames.

## Structure: the seamless-loop skeleton

```
frames 0..~40    ENTRANCE  (pop + draw-ons, staggered)
frames ~40..~140 HOLD      (breathing pulse, ripples, shine sweeps, shakes — all boundary-neutral)
frames ~140..178 EXIT      (stroke exits below, reverse story order)
frame  178 (op-2) EMPTY    → frame 0 empty ⇒ loops seamlessly; op:180 (6 s)
```

- Users trim to ~2–4 s in Camtasia for a hold ending (cut before the exit), or keep full length for the self-contained loop.
- Exit in **reverse story order** (bulb: light off → rays retract → filament drains → glass whips away).
- Final "empty/at rest" keys land at `op-2` (last playable frame), never `op-1`.

## Entrance recipes

### Draw-on (the signature move)
Path + trim + round-cap stroke. Group item order: `sh/el/rc, tm, st, tr`.
```
tm: s:0, e: 0→100 over 10–24f (SWISH), o:0, m:1
```
For circles start at 12 o'clock: static `o:-90`. TechSmith's own Checkmark: 3 keys `0 → 17.6 → 100` at t=0.5/10.5/20.5 with hard swish — fractional times are fine.

### Pop-in with rotation settle
On a **null parent** (`ty:3`, opacity 0 — children only inherit transform, not opacity):
scale `0 → 112 → 100` (0/10/16) + rotation `-14 → 4 → 0`. Feels physical.

### Burst (TechSmith Burst 1 anatomy)
One rect/tick + trim + repeater, and the tick **flies via position keys**:
```
gr [ gr Tick [ rc(p: (0,-290)→(0,-368) SWISH), tm(e: 0→100 over 10f; s chases for one-shot), fl, tr ], rp(c:8..24, tr.r: 360/c), tr ]
```
Trim without the position animation just shrinks the tick in place — always pair them.
Companion "ring pop": ellipse stroke with **width animating → 0** while group scale grows (Burst's 608px ring: w 40→0 over ~21f).

### Particle dots
Dot group with animated `tr.p` (radially outward) + `tr.s` (100→0), wrapped in a repeater; rotate the outer group 22.5° to interleave with other radial elements. Scale-to-zero beats opacity fades over arbitrary backgrounds.

### Light-on / state fill
Solid fill opacity `0 → 100` in ~10 frames. Avoid resting at partial opacity over unknown backgrounds (muddy). Put the fill group last in the layer's shape list so strokes render on top.

## Hold recipes (boundary-neutral, safe to trim through)

- **Breathing**: scale `100 ↔ 103` (badge) or `100 ↔ 107` (rays), SMOOTH, 16–30 frames per half-cycle; start and end at 100.
- **Ripple ring**: separate layer gated by `ip`/`op` (e.g. 60→96): ellipse stroke, group scale `100→148`, stroke width `18→0`, SWISH. Emit every ~60 frames. Rendered behind the badge it appears to emanate from under it.
- **Shine sweep** (needs alpha matte): matte-source layer = white copy of the plate shape with `td:1` placed immediately above consumer; consumer = rotated (~22°) rounded strip `150×980 r:75`, fill white at `o:26`, `tt:1`, position sweeping `x: -560 → 560` over 16 frames. Gate both layers with matching `ip`/`op` windows and repeat the pair for each pass.
- **Attention shake**: rotation wiggle on the null rig (see timing table). Pair with a subtle scale dip.
- **Baked infinite loop** (Iso-Ring pattern): for spinners/dials meant to run long, bake N cycles as explicit keys (`0/360/720/...` every 60 frames, 61 keys over 120 s verified) — never expressions. Author generously long; users trim down.

## Exit recipes — stroke-based "leave" vocabulary

Fading out works but reads as a shrug. Make strokes *leave*:

| Exit | Trim mechanics | Reads as | Best for |
|---|---|---|---|
| **Drain / run-off** | `s: 0→100` (SWISH), `e` stays 100 | stroke keeps moving forward, empties off its own tail | checkmarks (swoosh exits off the tip), letters (drain in writing direction), filaments, bars |
| **Whip-around** | `e: 100→0` while `o` spins a full turn (e.g. `-90 → -450`), same 14–18f window | closed outline collapses to an arc that whips once around and vanishes | circles, rings, octagon rims, bulb glass |
| **Retract** | `e: 100→0` + reverse any position flight | the draw-on played backwards | rays, underlines, anything that "grew out" |
| **Anticipation pop-out** | scale `100 → 106 → 0` on the layer/null | fill objects can't stroke-exit | badges, plates (after their strokes have left) |

Stagger exits in reverse draw order (letters: P → O → T → S, 4 frames apart, ~10 frames each).

## Glyphs and letters without text layers

Text layers are unsupported — build words as stroked paths, one group per letter with its own trim + stroke (`w:26`, white, round caps), positioned via the group `tr.p`:
- Blocky sign-font letters from straight segments work and stay legible (S as 6-point zigzag bar-form; T as two subpaths in one group — trim m:1 draws both simultaneously).
- O = ellipse primitive; P = stem + bezier bump (tangents ≈ 0.55·r).
- Letter cell: height 140, width ≤ 80, centers 150 apart fits "STOP" inside a 340-radius octagon.

## Matte, rig, and layer-order rules

- Matte source (`td:1`) must sit **immediately above** its consumer (`tt:1`); pair them per effect pass; parent both to the same rig so they transform together.
- Null rigs carry pop/shake/settle/breathing for a whole multi-layer design; give children `p:[0,0,0]` and `parent: <null ind>`. Null opacity stays 0 (not inherited).
- Within a layer, earlier shape groups render on top; within the timeline, lower `ind` renders on top.
- Every `gr` needs a trailing `tr` item — builders should append it automatically.

## Corpus crib index (known-working reference files)

Local corpus (this machine): `.../MCP Tests/Lottie-Training/`
`Harvested CSAssets Lottie Files/` = 290 TechSmith-shipped assets; `Works_with_rlottie/` = 52 verified third-party files.

| Want | Crib from |
|---|---|
| Draw-on stroke + swish easing | `Checkmark 1__Checkmark 1.json` |
| Radial burst + width→0 ring + layer stagger | `Burst 1__Burst 1.json` |
| Alpha mattes (incl. inverted) + trim offsets | `Touch Tap__Touch Tap.json` |
| Precomp + merge paths + path morph | `Slider 0-100__Slider 0-100.json` |
| Baked 120 s loop, repeater dial, hard blink | `Iso-Ring 1__Dial-001-Blue.json` |
| Click press dip | `Arrow Click Mac 1__Arrow Click Mac 1.json` |
| Marker highlight sweep (non-square canvas) | `Highlight 1__Highlight 1.json` |
| Gradients | `Works_with_rlottie/instagram.json` |
| Masks + complex logo build | `Works_with_rlottie/gmail (1).json` |
| Full seamless-loop skeleton + all three stroke exits | `Lottie-Training/checkmark-circle-draw.json`, `stop-sign.json`, `lightbulb-idea.json` (generator: `lottie_kit.py` patterns, see `lottie_kit.md`) |
