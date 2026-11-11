# Design guide, part 2: light grounds, finishing, transparency

## Light grounds — cut marks OUT of paper, never add glow TO it

A light background is not a dark one re-palettized. It is the opposite
construction, and getting this wrong is why light attempts come out muddy or
blown out. Measured across a set of light backgrounds that shipped as good:

- **"Light" is NOT high average luminance.** One of the best in the set averaged
  only **0.58** luminance with 28.6% of every frame below 0.35 — darker on average
  than several shaders in the *dark* library. It still reads as light. Mean
  luminance is the wrong metric; don't tune against it.
- **The tell is the brightest 5% of pixels sitting at 0.91–1.00.** That is bare
  paper showing through between the marks. The ground is near-white and always
  visible. Verify this percentile, not the mean.
- **Every mark is SUBTRACTIVE — the one non-negotiable.**
  `col *= mix(vec3(1.0), ink, coverage);` Light is *removed* from paper. Dark
  backgrounds add glow to near-black; light backgrounds multiply ink into white.
  Because a multiply can never exceed the paper, the result **cannot blow out** —
  which is why the genre needs no tonemap and why additive glow instantly ruins it.
- **Content protection works UPWARD.** Lift the center back toward the paper color
  (`col += (_Paper.rgb - col) * 0.075 * smoothstep(1.30, 0.05, rad);`), or shrink
  and dim the marks there. Do NOT reach for a vignette — darkening the edges of a
  light ground reads as dirt on the page.
- **Finishing is nearly absent.** No tonemap (nothing approaches clipping). Grain
  only, at `0.008–0.015`, to stop the broad pale flats banding. Little to no
  vignette. This is the flat/bold end of the finishing spectrum.
- Motion stays as slow as anywhere else: internal time scale `0.10–0.12` before
  the user's `Speed`.

Palette shape that works: a near-white paper (`FFFFFF`, or a cream like `F2E9D0`),
then either **pale tints** (value 0.85–0.95, low saturation) for a calm editorial
look, or a few **saturated inks used as small discrete marks** for a bold print
look — never a saturated wash, which turns the page muddy.

### Layered planes with cast shadows (the "Cut Paper" recipe)

A light-ground genre that reads expensive and frames content unusually well:
pale paper planes floating at different heights over a page, each casting a real
soft shadow on the ones below. Depth comes from **occlusion**, not from color or
glow — which is what distinguishes it from flat shape collage. The shape
vocabulary is Memphis/postmodern-adjacent (also the family Google's Material
illustration style draws from): generous rounded rectangles and discs, pale
tints, slight rotations.

- **Painter's algorithm in ONE loop, shadow-then-fill, back to front.** This is
  the whole trick. Per layer: draw its shadow (multiply), then its opaque fill.
  A plane's shadow lands on everything already drawn, its own fill covers that
  shadow, and the next plane up covers both. No second pass, no z-buffer.
- **Order the layout table by ascending height so loop order IS stacking order.**
- **Physical penumbra — move all three together.** As a plane's height rises, its
  shadow is thrown *further*, blurs *wider*, AND gets *fainter* (the same light
  spread over more area): `off = h*0.16*_Elevation;`
  `blur = (0.012 + h*0.085)*_Elevation + aa;` `str = _ShadowStrength*(1.0 - 0.34*h);`
  Offsetting alone reads as a sticker with a drop shadow.
- **Ease the SDF shadow ramp.** `sh = 1.0 - smoothstep(-blur, blur, sds);` is
  linear and reads synthetic; `sh = sh*sh*(3.0-2.0*sh);` makes it fall off softly.
- **Contact shadow seats the plane.** A tight occlusion band hugging the edge on
  *all* sides, independent of the thrown shadow:
  `ao = 1.0 - smoothstep(0.0, 0.012 + 0.030*h, sdBody);` at ~0.20 strength. Without
  it every plane looks pasted on, no matter how good the cast shadow is.
- **A lit rim gives the sheet thickness.** A hairline catch of light just inside
  the boundary, only on the light-facing side:
  `rim = smoothstep(-0.024, -0.005, sdBody); ls = clamp(dot(pn, lightDir)*1.4, 0.0, 1.0);`
  then `fill += (vec3(1.0) - fill) * 0.34 * rim * ls;`. Turns a flat vector shape
  into cut paper.
- **Tint the shadows toward the paper casting them** (~28% toward that plane's own
  color), never neutral grey.
- **Atmospheric perspective:** wash lower planes further toward the page
  (`mix(tint, _Paper.rgb, 0.42 - 0.20*h)`), so the stack reads as depth, not collage.
- **Hand-place the layout, don't hash it.** Which shapes overlap and where the frame
  stays open is a composition decision a `fract(sin())` hash cannot art-direct.
  Anchor every plane past the frame edge or in a corner, keep the center open, and
  cap drift amplitude so no plane can wander into it.
- **Use enough planes.** Six forced each one to be enormous and the result read as
  a flat UI mockup; **nine** smaller ones read as a composed collage, and the size
  range itself carries the depth.
- One primitive covers everything: a rounded box whose half-extents equal its
  corner radius *is* a disc, so `roundBox` alone draws the whole vocabulary.

## The finishing pass (the step most people skip)

Raw additive layers clip to ugly white and band on 8-bit displays. Every great
file ends with some subset of these, in order:

1. **Tonemap** to roll off highlights instead of clipping:
   - Reinhard-ish: `col = col/(1.0 + col*k)`, `k≈0.12–0.6` (higher = more
     compression, for busier/brighter looks).
   - Filmic: `col = col/(col + X); col = pow(col, vec3(0.82));`, `X≈0.9–1.2`.
2. **Grain / dither** to kill gradient banding:
   `col += (hash(uv*_ImageSize + fract(_Time)*k) - 0.5) * A;` with `A≈0.012–0.016`.
   Small, per-pixel, lightly animated. This one line is the difference between
   "smooth" and "visibly stepped."
3. **Saturation recovery** — tonemapping desaturates, so pull it back:
   `float lum = dot(col, vec3(0.299,0.587,0.114)); col = mix(vec3(lum), col, 1.3);`
   (≈1.25–1.5). Essential for neon / brand looks.
4. **Vignette** as a final multiply:
   `col *= mix(1.0, smoothstep(1.8,0.35,length(p)), _Vignette);` — darkens edges,
   focuses the eye, and further protects overlaid content.
5. Optional: nudge the brightest pixels slightly cool
   (`col.b += (1.0-col.b)*0.04`) to avoid a warm-clipped highlight.

Bold/flat genres (hard-edged rays, cartoon bursts) deliberately skip most of
this — few colors, sharp edges, no tonemap. Match the finishing to the genre.

## Transparency & alpha output (premultiplied!)

Backgrounds are usually opaque (`_FragColor = vec4(col, 1.0)`), but sometimes you
want *part* of the shader transparent so lower timeline tracks show through — e.g.
a "paper opacity" knob that fades a solid base to nothing while the motif stays
solid, letting the effect float as an overlay.

**Camtasia and CamRust composite shader output with PREMULTIPLIED alpha.** So you must scale
RGB by alpha in the output:
```glsl
float alpha = /* 0..1, e.g. max(motifMask, _BaseOpacity) */;
_FragColor = vec4(clamp(col,0.0,1.0) * alpha, alpha);   // premultiplied — RGB * alpha
```
Emit straight (non-premultiplied) `vec4(col, alpha)` and the transparent regions
**add** their color onto the layer below and blow out to white — a cream base
(≈0.95) over a blue track clips to `(1,1,1)`. That *white-instead-of-showthrough*
is the tell-tale signature of a missing premultiply; if a transparency knob makes
gaps go white, this is why. At `alpha = 1` everywhere, `col*1 == col`, so an
opaque default is unchanged — premultiplying is free insurance even when the
default look is fully opaque.

Expose the control as a `0..1` float with `"unitType": "percentage"` so the panel
reads 0–100%. Keep the motif's own pixels at `alpha = 1` (via a coverage/SDF mask)
so only the *base* turns transparent, not the thing the user came for.
