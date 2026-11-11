# Design guide, part 1: build recipe, motion, element fields, palette

Distilled from shaders that shipped as "great"; the numbers are the ones those files use, so treat them as calibrated starting points. Part 2 (`design-light-finishing.md`) covers light grounds, the finishing pass and transparency.

Technical correctness is table stakes. A dynamic background succeeds or fails on
aesthetics. Everything below is distilled from a body of shaders that shipped as
"great" — the numbers are the ones those files actually use, so treat them as
calibrated starting points, not guesses.

**It's a supporting actor.** Titles, callouts, and talking heads sit ON TOP of
it. The background adds polish and mood without competing — "seasoning, not the
meal." Two tests before you ship: would white AND black text be readable over
most of the frame, and does anything rhythmically grab the eye? If yes to the
second, calm it down.

## The layered build recipe

Almost every great background is the same four-layer stack, composited
back-to-front inside one `main()`. Build in this order and you rarely go wrong:

1. **Base gradient** — never a flat fill. Two colors (a deep shade + a slightly
   lighter tint) mixed along a *diagonal*, via a dot product with a normalized
   diagonal vector:
   `float g = dot(uv, normalize(vec2(0.7,0.5)))*0.35 + 0.5; col = mix(deep, tint, clamp(g,0.,1.));`
   Diagonals read more designed than vertical/horizontal ramps.
2. **Corner light source(s)** — 1–3 soft additive glows placed at *specific*
   off-center points. This is what makes a scene feel *lit* rather than filled.
   Anchor them in corners/edges so the center stays clean (falloff rules below).
3. **Element layer(s)** — the actual motif (blobs, particles, lines, rings,
   rays…), drawn back-to-front with parallax if there is more than one. This is
   the only layer the user reads as "the effect"; everything else is atmosphere.
4. **Finishing pass** — tonemap → grain → saturation recovery → vignette.
   Non-negotiable for a professional result (see *The finishing pass* below).

`references/starter.tscshadervid` is a minimal, valid file wired to exactly this
stack (6 properties, full finishing pass). Copy it to start instead of retyping
the boilerplate — but it is a **skeleton, not a template**: replace layer 3 with
a real motif from the cookbook and re-palette it. Never ship its placeholder
blobs, or every background ends up the same.

## Composition — protect the center

- **Anchor interest at corners and edges; keep the center calm.** Overlaid
  content lands center-frame, so bias glows, density, and brightness outward.
- **Give the user a calm knob.** The best files expose composition as a
  property: a `CenterCalm`/`EdgeBias` that fades element brightness toward the
  middle (`mix(1.-calm, 1., smoothstep(0.15,0.95,length(uv)))`), a one-sided
  `rightMask` that pushes energy to one side and leaves the other clear for
  text, or a `Vignette`. Let users defend their text without editing GLSL.
- **Anchor moving elements OFF-CANVAS.** Blob centers, ring origins, and tail
  pivots live at ±1.5 and beyond (outside the visible ±aspect frame). Nothing is
  "born" on screen — elements only drift partway in, so there are no visible
  spawns or hard entering edges.

## Motion — slow, ambient, layered

- **Slower than you think.** These files scale `_Time` by a small internal
  constant *before* the user's `_Speed`: ~0.05–0.15 is typical (≈0.045 for the
  calmest ripples, 0.12–0.15 for aurora/fire). Drift and orbit frequencies sit
  around 0.02–0.11. Elements should take *tens of seconds* to cross the frame.
- **Incommensurate frequencies.** Layer sines at unrelated rates (0.05, 0.07,
  0.09, 0.11…) so motion never visibly repeats. A single dominant period reads
  as a metronome.
- **Gentle pulses, never on/off.** Breathing is `0.5+0.5*sin(...)` or
  `0.7+0.3*sin(...)`, not a hard blink. Every element gets a random phase from a
  hash so nothing ticks in unison. Vary rhythm and speed like layers in a mix.
- **Fast is a deliberate style, not a default.** A "digital"/tech look can
  flicker fast (one grid runs 4–6.5 Hz) — but even then fade each change and put
  a slow global breath underneath. If you go fast, let the genre justify it.
- **In-place animation alone reads as static; a slow global drift is what sells
  "alive".** Elements that light up, pulse, or breathe *in place* often still feel
  like a frozen still with some twinkle — a real "it's not moving" complaint. The
  fix is to move the whole field slowly through space. For any coordinate-derived
  pattern (grids, circuits, fields, terrains) this is nearly free: offset the
  sample coordinate by time — `gp = uv*cells + _Time*_PanSpeed*dir` — and new
  procedural content scrolls in **forever, seamlessly**, an infinite camera pan.
  A gentle diagonal reads best (`normalize(vec2(1.0, 0.62))`); ~0.3 cells/sec
  (≈30 s to cross frame) is a calm default.
- **Layer motions on INDEPENDENT sliders, and keep lighting screen-fixed.** Give
  each motion its own control at `0`-means-off: a global `PanSpeed`, an
  element/packet motion, a breathe. Independent axes hand the user a
  combinatorial set of looks for free (locked board + flowing data; slow flyover
  with calm packets; fast scroll for an intro) and let each freeze separately for
  a clean static frame. Keep the pan in *pattern space* (offset `gp`) but vignette,
  center-calm, and gradients in *screen space* (the un-offset `uv`/`p`), so the
  camera pans over the board while the lighting stays anchored to the frame.

## Depth & parallax

Multiple layers with *correlated* size/speed/opacity sell depth: bigger =
nearer = faster = brighter; smaller/dimmer = farther. Concretely — geometric
size ratios per layer (`pow(0.6, i)`), speed tied to size (`speed ∝ size^0.75`),
2–6 layers is plenty; a 20–30% speed difference between layers already reads.
Soften edges proportionally to element size and dim/desaturate the distant layer
(atmospheric perspective). Even a near + far pair beats a single flat layer.

## Variation kills the mechanical feel

Randomize per-element size, opacity, offset, and timing from a hash. If elements
come from a grid, jitter their positions until the grid is invisible.

## Discrete-element fields on a hashed grid (bokeh, dots, nodes, rings)

Any field of many independent elements — bokeh discs, particles, ring-outlines,
constellation nodes — is best built from a hashed cell grid, not a fixed list.
The pattern hands you the design knobs for free:

- **Cells + neighbourhood:** `gp = p*scale; cell = floor(gp);` then loop a small
  neighbourhood (`for oy=-2..2, ox=-2..2`) so elements straddle cell borders. 3×3
  for small elements; 5×5 when one element can span >1 cell (or a `RingSize`
  slider can grow it past a cell).
- **Density = a host threshold, not a count.** Gate each cell on a hash:
  `if (hash21(cid*1.7) < hostThresh) continue;`. A user `Density` slider just
  remaps `hostThresh` (higher → fewer elements). This is the concrete lever
  behind *Density & scale* — it's how you escape the "visual static" trap without
  touching geometry. A first pass almost always comes out too dense; raise the
  threshold hard before anything else.
- **Per-element variety from the cell id:** jittered centre
  `cid+0.5+(hash22(cid)-0.5)*k`, plus independent hashes for radius, colour
  category, brightness, and pulse phase — so no two elements match and nothing
  ticks in unison.
- **Ring / annulus outline primitive** (beyond the disc/blob vocabulary): a crisp
  AA'd band plus a tight bloom around the radius —
  `edge = abs(length(gp-c)-radius); band = 1.0 - smoothstep(halfw-aa, halfw+aa, edge);
  glow = exp(-edge*edge*K);`. Make the stroke *proportional* so a `Thickness` knob
  reads at every size: `halfw = max(aa*1.2, radius*0.055*_Thickness)`.
- **Analytic AA without `fwidth`:** derive it from the cell scale —
  `pxPerUnit = _ImageSize.y*0.5/scale; aa = 1.6/pxPerUnit;`. Scales correctly per
  layer, no derivative function needed.
- **Keep bloom TIGHT or the darks die.** Stacking several additive element layers,
  each with a soft glow, silently accumulates into a grey haze that flattens a
  near-black base — the #1 thing separating "professional" from "murky" in an
  element field. Push the bloom exponent high (`exp(-edge²*~90)`, not `~40`), scale
  per-element bloom down, and reserve the broad *linear* glow for the 1–3 corner
  light sources only. Confirm the base corners still read near-black in the FINAL
  composited render, not just in the base-only pass.
- **Drift each layer independently** (see *Measure drift by shift-search*): a
  `vec2` velocity per layer in *mixed* directions + a slow shared meander
  (`gp += vec2(sin(t*0.5+scale), cos(t*0.42-scale))*0.12;`) floats organically; a
  single shared velocity reads as a conveyor belt.

## Connected structures on a grid (circuits, mazes, pipes, roads)

When the motif must *connect continuously* across cells — PCB traces, maze walls,
pipe/road networks — the trick is to hash the **shared boundary, not the cell**:

- **Shared-edge hashing = continuity for free, with NO neighbour loop.** Key each
  edge's on/off state to the edge's own integer coordinate, so both cells sharing
  it read the identical value. A cell's west edge is `edgeOn(vec2(cx, cy))`, its
  east edge `edgeOn(vec2(cx+1, cy))` — the neighbour's west edge is the *same*
  call, so traces always line up at the border. Each fragment then only evaluates
  its OWN cell (draw segments from cell-centre to each active edge midpoint), yet
  the paths read as long continuous branching runs. This is far cheaper than the
  neighbourhood loop the *independent-element* grid needs.
- **`smin` is a one-knob corner/junction rounder.** Smooth-union the centre→edge
  segments (`d = smin(d, segSDF(f, C, midpt), k)`) and a single `k` rounds
  corners, T-junctions, AND crossings uniformly — no per-orientation arc maths.
  `k=0` → hard right angles; larger `k` → rounder. Expose it as `CornerRadius`.
  (`smin(a,b,k)= mix(b,a,h)-k*h*(1-h)`, `h=clamp(0.5+0.5*(b-a)/k,0,1)`; guard
  `k<=0` → plain `min`.)
- **Count the active edges to place furniture:** `count = N+E+S+W` → draw a via/pad
  donut at junctions (`count>=3`), an end-pin at stubs (`count==1`), nothing at
  straight-throughs/corners. Reads instantly as a real board.
- **A low-frequency region mask buys negative space** — the openness this genre
  needs for overlaid text. Uniformly lowering density just thins everything into a
  sparse even sprinkle; instead multiply the activation threshold by a smooth
  value-noise field keyed off the *same shared id*
  (`dens * smoothstep(0.32,0.68, vnoise(id*0.14))`) so whole regions go empty while
  others stay dense — designed clustering, not uniform coverage, and continuity is
  preserved because the mask is a function of the shared id.

## Density & scale — the calmness lever

For any repeated-element motif (dots, tiles, particles, a grid), **fewer + larger
reads far calmer than many + smaller**, even at the same coverage. Small dense
elements at 1080p degrade into visual static — an eye-test chart the eye can't
rest on — no matter how tasteful the palette. The instinct to "fill the frame" by
shrinking elements backfires; instead use *bigger* elements and *lower* coverage
so the base shows through between them. The result still fills the frame (no empty
gaps) but reads as an intentional, restful pattern. When a busy motif won't calm
down, scale the elements UP and thin them out before touching anything else.

- **The recipe:** two background colors (deep + tint) for the gradient, one or
  two element colors, and one accent that behaves as a *light source*. One hue
  family plus one accent; gradients over flat fills throughout.
- **Accent as light, not decal.** Place the accent as glows that tint and
  brighten nearby elements, so the scene feels lit by it. That cohesion is what
  reads as "designed."
- **Match the falloff to the job** (this corrects a common oversimplification —
  it is not "always linear"):
  - **Linear `exp(-d*k)`** — broad, reaches across the frame (`k≈0.5–1.3`). Use
    for *ambient* corner light and large washes. This is the light-source
    falloff.
  - **Gaussian `exp(-d*d*k)`** / `exp(-dot(v,v)/r²)` — compact, self-contained.
    Use for *localized* blooms: blob cores, motes, point glows, node halos. It
    stays put instead of graying the whole frame.
  Reach for gaussian when a glow should stay small and crisp; linear when it
  should fill space.
- **Dark palettes flatter overlaid white text and survive projection.** Most
  great backgrounds are dark (deep navy / near-black bases). Bright or light
  palettes work, but they are a different construction, not the same shader with
  the colors swapped — see *Light grounds* below for the mechanism. Tiny
  near-white blooms are fine; large blown-out regions are not.
- **Watch hue interactions.** A saturated element color laid over a
  differently-hued bright field muddies (e.g. green over a bright teal field
  skews yellow). Keep each brand color where it can stay pure — in a calm/dark
  zone, or as discrete accents rather than a wash — and reserve one hue per
  region.

