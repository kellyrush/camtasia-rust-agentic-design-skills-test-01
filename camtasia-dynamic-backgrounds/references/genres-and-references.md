# Panel design, genres, anti-convergence and reference matching

## Property / UX design IS design

The panel is part of the deliverable. Conventions from the great files:
- **6–10 properties.** Colors first (2–5), then `Speed`, then look-shaping
  scalars (`Density`, `Glow`, `Size`, `Amount`), then composition knobs
  (`CenterCalm`, `Vignette`, angle dials) last.
- **Almost always present:** a `Speed`, a `Glow`, and a density/amount control.
- **Ranges so the default sits mid-range and both extremes stay usable.** Verify
  min AND max of every slider still looks intentional.
- **Signed speed** (`-3..3`, or degrees/sec) where reversing direction is
  meaningful (rotation, drift).
- **The DEFAULTS are the deliverable** (see the authoring workflow). The
  instant-drop look must be the calm, professional one; sliders reach the
  energetic extreme, not the reverse.

- **Match energy to purpose.** Instructional/corporate wants soft, gradual
  motion; promo/intro can run bolder and faster.
- **Speed = 0 should still be a good background.** A well-composed frozen frame
  proves composition and color stand alone, and serves motion-sensitive viewers.
  Keep amplitude low regardless — full-frame swings and zooms trigger motion
  sickness; small drifts do not.
- **Loops and long watches.** Shader time is unbounded; avoid short obvious
  periodicity (a pulse every 2s reads as a metronome). Layered incommensurate
  frequencies feel organic for minutes.

## Genre → technique cookbook

Recognize (and be able to propose) the recurring genres. Each maps to a concrete
GLSL primitive — so once you've named the look, you know how to build it:

| Genre | Look | Core primitive |
|-------|------|----------------|
| **Aurora / mesh gradient** | soft drifting blobs of 2–3 hues; the "modern SaaS" default | FBM domain-warping, or a few gaussian blobs `exp(-dot(d,d)/r²)` drifting on incommensurate sines |
| **Fluid / organic** | smoke, silk, ink, flame ribbons, curling tails; luxurious | layered FBM noise; sine "ribbons" `exp(-abs(y-wave)/thick)`; keep contrast low or it overcooks |
| **Bokeh / particle field** | soft discs/shapes at varied size & opacity, parallax drift; elegant, calm | hashed cells (`floor`/`fract` grid) with per-cell jitter/size/phase; translucent, back-to-front |
| **Geometric / minimal** | sparse lines, bars, grids, tiles; precision & restraint (brand-forward) | SDFs (rounded-box `length(max(abs(p)-b,0.))`, triangle); static or step-and-pause motion |
| **Network / constellation** | nodes + connecting lines, "knowledge graph" feel | hashed node positions + `segDist` links between near pairs; pulse the nodes |
| **Ripple / rings** | concentric waves from off-canvas origins | `sin(length(uv-origin)*density - t)` gated by `exp(-dist)`, noise-perturbed |
| **Light rays / streaks / tunnels** | directional energy, depth flythroughs; the boldest end, best for short intros | polar coords (`atan`,`length`); nested SDF rings + exponential radius for tunnels; angular repeat for bursts; flow-field advection for streaks |
| **Layered planes / cut paper** | pale geometric sheets floating over a page, real soft cast shadows; Memphis/Material lineage, premium & light | hand-placed `roundBox` layout table ordered by height; one loop, shadow-then-fill back-to-front; penumbra offset+blur+fade all scale with height (see *Light grounds*) |
| **Light & shadow / gobo** | dappled foliage shadow on a sunlit wall; no marks at all, one plane unevenly lit | two thresholded FBM tiers (broad distant + crisp near) multiplied; ridged noise to scallop the edges; warm light color vs cool shadow color, both multiplied into paper |

Rule of thumb the set confirms: **calm/atmospheric looks are dark, layered, and
heavily tonemapped; bold/flat looks (hard-edged rays, cartoon bursts) use fewer
colors, sharp edges, and little-to-no tonemap.** Match the finishing to the genre.

**Reference material from the user is the best input.** If the user offers
screenshots or videos of backgrounds they like — or you can ask for some — take
them: extracting palette, motion, density, and mood from real references (see
workflow below) beats designing from adjectives every time.

Inspiration sources beyond the obvious: Shadertoy (technique + look),
web UI background libraries ([reactbits](https://reactbits.dev/backgrounds/aurora),
[shadcn backgrounds](https://www.shadcn.io/background),
[Aceternity](https://ui.aceternity.com/components/aurora-background)) for
current web aesthetics, VJ-loop and stock marketplaces (Envato "VJ loops",
Storyblocks) for broadcast/stage energy, and Dribbble/Behance motion tags for
curated taste. Broadcast/stage design has a useful framing: lighting is the
*atmosphere layer* that guides attention; content is the *message layer* — a
good background is atmosphere, not message.

Sources: [Smashing Magazine — Ambient Animations](https://www.smashingmagazine.com/2025/09/ambient-animations-web-design-principles-implementation/),
[TechSmith — Motion Backgrounds](https://www.techsmith.com/blog/make-motion-background-for-video/),
[web.dev — prefers-reduced-motion](https://web.dev/articles/prefers-reduced-motion).

## Anti-convergence: derive the style, don't default to one

A skill like this accumulates gravity toward its own past: worked examples,
shaders already built with it, and famous genres (aurora blobs, particle waves,
retro grids) start to feel like "the answer". Left unchecked, every open-ended
brief — "match our branding", "make it feel like this URL/doc" — converges on
the same two or three looks. Rules for open-ended briefs:

1. **Evidence before genre.** Extract concrete signals FIRST and write them
   down: palette (sample real pixels from the brand kit / URL / screenshots),
   shape language (rounded vs sharp vs organic), density, texture, typography
   weight, motion personality (calm / technical / playful / premium), industry
   tone. Only after that ask which visual treatments — plural — fit the
   evidence. Never pick a genre before the evidence exists.
2. **Offer 2–3 genuinely different directions**, not variants of one genre,
   and let the user choose. At least one should be a style you have NOT built
   recently. A brand with sharp engineering geometry is usually better served
   by geometric-minimal or grid treatments than by another soft aurora; a
   playful consumer brand may want bold flat shapes, not bokeh.
3. **Audit recent output.** Look for previously built shaders (nearby
   `.tscshadervid` files, the user's media library) and steer away from
   repeating their genre and palette unless the user explicitly wants a
   matching set.
4. **Worked examples are format references, not taste references.** Copy their
   file structure, property patterns, and QA loop — never their look. The
   genre cookbook above is for *naming* options, not a menu of defaults;
   treat its genres as starting points to recombine or depart from.

## Matching a reference video or image

1. **If matching a reference** (video/image): sample frames with ffmpeg across the
   duration (`ffmpeg -ss T -i in.mp4 -frames:v 1 out.jpg`) and read them. Identify
   the *procedural structure* (e.g. "nested triangle tunnel = repeating rings +
   exponential radius + per-ring twist + depth-gradient + core flare + starfield"),
   not literal pixels. Decide which knobs a user would want. Checklist:
   - **Check for looping first.** Sample a cheap statistic (e.g. mean R/G/B of a
     corner region) every few seconds across the WHOLE video — an exact repeat
     reveals the loop period. Only the unique span needs analysis, and a known
     loop period sanity-checks any motion/speed estimates.
   - **Temporal-min composite isolates the background.** When the moving elements
     only *lighten* the frame (additive/screen particles, glows), a per-pixel MIN
     across ~30 frames spread over the full duration recovers the bare background
     almost perfectly. This cleanly separates "background gradient" from "what the
     elements contribute" — and can reveal that an apparent background feature
     (e.g. a corner glow) is really the elements being tinted.
   - **Symmetry hides motion.** A square at +30° looks identical at −60°; sparse
     samples can't distinguish continuous rotation from step-rotate-pause, or CW
     from CCW. After the coarse pass, pull enlarged nearest-neighbor crops
     (`crop=…,scale=…:flags=neighbor`) at *fine* time steps around transitions to
     pin down direction, easing, dwells — and secondary motion users notice
     (overshoot/settle wobble, motion-blur softening mid-move).
   - **Separate hue from brightness for gradients.** Heavily blur a frame, then
     view it hue-normalized (divide each pixel by its max channel). This exposes
     the gradient's structure at a glance. Don't assume linear — test corner/edge
     samples against the model before coding; corner-pinned radial blobs + a
     soft center glow fit many real backgrounds.
   - **Simple beats clever on compressed video.** Phase correlation, template
     rotation matching, and structure-tensor angle estimates all misled on mp4
     artifacts; big crops + eyeballs won, plus autocorrelation of a high-passed
     strip for grid pitch. Sample only clean regions — mask text/overlays.
   - **Measure drift by shift-search, and calibrate the sign on your OWN
     render.** For a translating field (bokeh, particles, snow), estimate
     velocity by minimizing MSE over integer row/col shifts between two frames a
     fixed gap apart (a tiny brute-force search beats optical flow here). Two
     traps this catches: (a) the sign is ambiguous until you calibrate it — run
     the SAME shift-search on a render whose drift direction you already know
     from the code, then trust the sign on the reference; (b) **a "drifting"
     stock background often has NO single global velocity** — measure at several
     points across the runtime and you may find the direction reverses (down
     early, up later). That means the elements *float and meander*, not ride a
     conveyor. Reproduce it with per-layer drift vectors in *mixed* directions
     plus a slow shared meander (see *Depth & parallax*), never one global
     velocity — which also removes the tell-tale "conveyor belt" read.

**Snapshot on user approval.** The moment the user approves a look, copy the
file before iterating further, and give the copy a distinct top-level `"name"`
so both versions can coexist in the media library. Later experiments then can't
destroy the approved version.

**Never change a default at the last second — the DEFAULTS are the deliverable.**
Users judge the shader by how it looks the instant they drop it on the timeline;
almost none will "dial it in" via properties, and they shouldn't have to. So the
default value of every property must reproduce the reference look on its own.
Two failure modes that have burned this skill twice:
- *Tuning a default against the wrong reference.* When matching a video, tune and
  verify defaults against **frames from that video**, not a separate "sample"
  image/PNG the user also provided — supplied stills are often graded richer or
  flatter than the actual footage, and chasing them pushes the default off (e.g.
  cranking a depth/contrast knob until rays look "deep enough" in the sample,
  leaving them too dark versus the real video). Use the extra reference for
  *structure* (edges, counts, layout); use the video for *color/contrast defaults*.
- *A "harmless" final edit that shifts the resting frame.* Any change late in the
  session — even one meant only to fix motion (a rotation-sign flip, a speed
  tweak) — can ride along with a default change. After the LAST edit, re-render
  the pure defaults (a fresh import renders them) and re-compare to the reference
  **before** declaring done. Don't hand off on the strength of an earlier
  comparison. If you change a default after the user last saw it, say so explicitly.
- *Adding a NEW property must be default-invariant.* When you extend an approved
  shader with a new knob (a `Thickness`, a `Wobble`), choose the default so the
  new term is an identity at that value — `radius*0.055*_Thickness` with
  `Thickness=1.0`, an additive wobble scaled by `_Wobble` with `Wobble=0.0` — and
  then PROVE it: render the pure defaults and pixel-diff against the last approved
  frame; they must be identical. Only after that, render a cranked-default copy to
  show the knob works. This lets you keep growing the panel without ever risking
  the look the user already signed off on.
