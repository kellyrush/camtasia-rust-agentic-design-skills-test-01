---
name: camtasia-dynamic-backgrounds
description: "For CamRust. Author custom Dynamic Backgrounds: .tscshadervid files holding one GLSL fragment shader plus user-adjustable properties (colors, sliders, angle dials). Use to create, edit, debug or match a reference animated shader background. Covers the file format, GLSL uniforms and outputs, the effectDef panel schema, design guidance, and a CamRust import-and-render QA loop."
author: OpenCode
---

# Dynamic Backgrounds (`.tscshadervid`) for CamRust

A `.tscshadervid` is a **single self-contained JSON file** holding one GLSL
fragment shader plus a list of user-adjustable properties. CamRust (like
Camtasia) treats it as ordinary media: `import_media` it, place it with
`add_media_to_timeline`, and the user gets a live, parametric **Dynamic
Background**. The WHOLE effect lives in one file; never layer several media to
fake it.

Files you author here should also open in Camtasia Editor, so the format and
pitfall rules below cover both.

## Golden rules

1. **One file = the whole effect.** Background, particles, lines, glow,
   gradient and animation are all computed inside one `main()`. If you are
   about to import several shaders to composite a look, fold it into one.
2. **Author the UNSIGNED form.** Shaders that ship with Camtasia are signed
   (`code` on one line with escaped `\n`, a `signature` block, sometimes
   `numVertices`/`vertexCode`). Do NOT imitate that. Hand-authored shaders use
   the plain multi-line form, literal newlines, no signature, no `vertexCode`
   (CamRust does not draw vertex/particle shaders yet).
3. **It loads like normal media.** `import_media` with the file path returns a
   `source_id` with kind `dynamic_background`, its `properties` (the effectDef
   names) and a `warning` if the GLSL won't compile.
4. **Bad GLSL draws nothing in CamRust, and can crash Camtasia Editor.** CamRust
   reports the compile error once (the import `warning`) and the clip renders
   transparent. Treat any import warning as a hard failure: fix it before
   anything else. Never hand a user a file that only "imports"; render it.

## Duration: extend the clip, never tile it

A shader is procedural and endless: there is no intrinsic duration to trim
down to. It is added for the default image duration unless you pass
`duration_s` (or `media_end_s`) to `add_media_to_timeline`; to cover a long
timeline, place ONE clip with `duration_s` = the full length, or change it later
with `set_property` (`duration_s`). Do not lay down copies with cross-faded
seams: it adds transitions, risks visible seams and multiplies clips for nothing.

## File format

```jsonc
{
  "name": "NeonTriangleTunnel",   // library display name; unique per version
  "version": 1.1,                  // 1.0 / 1.1 both fine for authored files
  "width": 1920,
  "height": 1080,
  "copyright": "…",               // optional
  "code": "                        // GLSL fragment shader, literal newlines OK
uniform vec4 _ColorA;              // one uniform per adjustable property
uniform float _Speed;
void main() {
   vec2 uv = (_TexCoord*2.-1.)*(_ImageSize/_ImageSize.y);
   _FragColor = vec4( … , 1.0 );   // write the output here
}
",
  "effectDef": [ /* one entry per uniform, minus the leading underscore */ ]
}
```

With literal newlines in `code` the file is **not strict JSON** (control
characters in a string). That is expected and matches every sample; both
parsers are lenient. Don't run `json.loads` on the whole file to validate it.

### GLSL contract (NOT ShaderToy)

| You write | Meaning | ShaderToy analog |
|-----------|---------|------------------|
| `void main()` | entry point | `mainImage(out vec4, in vec2)` |
| `_TexCoord` | vec2 UV in `[0,1]`, **y-down** (0 = top row) | `fragCoord/iResolution` |
| `_ImageSize` | vec2 resolution in px | `iResolution` |
| `_Time` | float seconds elapsed | `iTime` |
| `_FragColor` | vec4 output, **premultiplied alpha** | `fragColor` |
| `uniform vec4 _Foo;` | a **Color** property | — |
| `uniform float _Foo;` | a **double** slider/dial | — |

Centered, aspect-correct coordinates (every sample uses this):
```glsl
vec2 uv = ( _TexCoord * 2.0 - 1.0 ) * ( _ImageSize / _ImageSize.y );
```
No `gl_FragColor`, `mainImage`, `fragCoord`, `iResolution`, `iTime`: those are
ShaderToy leftovers and won't compile here.

### `effectDef`: the property panel

Every `uniform _Foo` needs exactly one entry named `Foo` (drop the `_`), and
vice versa. Three property kinds:

```jsonc
// Color -> vec4 uniform. Hex string, no '#'. Alpha implied 1.0.
{ "type": "Color", "name": "ColorA", "value": "18E0D8" }

// double (slider) -> float uniform. Two accepted spellings:
{ "type": "double", "name": "Speed", "minValue": 0, "maxValue": 5, "defaultValue": 1.2 }
{ "name": "Speed", "type": "double",
  "valueBounds": { "minValue": 0, "maxValue": 5, "defaultValue": 1.2 } }

// directional dial (angle, no min/max) -> float uniform (radians)
{ "type": "double", "name": "Angle", "defaultValue": 0.3, "userInterfaceType": "directional" }
```

Optional: `"unitType": "percentage"` on a 0..1 double so the panel reads 0–100%.
Keep the panel usable: **6–11 properties**.

Naming: prefer generic labels for user-assignable colors (`Color1`, `Accent`,
not `ColorGreen`). **Trailing digits display shifted +1 in Camtasia's panel**
(`Color1` shows as "Color 2"; the digit is treated as a 0-based index). If the
label matters, name one lower (`Color0` → "Color 1") or avoid trailing digits.

## Parity checklist (run by eye before every import)

1. Every `uniform _X` has one effectDef entry `X`, and every entry has a uniform.
2. Types agree: `Color` ↔ `vec4`, `double` ↔ `float`.
3. `code` contains `void main`, `_TexCoord`, `_ImageSize`, `_Time`, `_FragColor`.
4. **No `"` anywhere inside `code`, not even in a `//` comment.** A stray quote
   ends the JSON string early and corrupts everything after it. Reword comments
   with dashes or parentheses.
5. No ShaderToy tokens (list above).
6. Property count ≤ ~11.
7. If you changed a property name, the import's `properties` list shows the new
   name (and no stale one).

## Design essentials

Technical correctness is table stakes; a background succeeds on aesthetics.
**It's a supporting actor**: titles, callouts and talking heads sit on top.
Would white AND black text read over most of the frame? Does anything
rhythmically grab the eye? If yes to the second, calm it down.

- **Four-layer stack, back to front, in one `main()`:** (1) base gradient along a
  diagonal, never a flat fill; (2) 1–3 soft corner light sources; (3) the element
  layer (the motif); (4) finishing pass: tonemap → grain → saturation recovery →
  vignette.
- **Protect the center.** Bias glows, density and brightness to corners and
  edges; anchor moving elements off-canvas (±1.5 and beyond) so nothing is born
  on screen; expose a calm knob (`CenterCalm`, `Vignette`, an edge bias).
- **Slow, ambient, layered motion.** Scale `_Time` by ~0.05–0.15 before the
  user's `_Speed`; layer sines at incommensurate rates; pulses are
  `0.5+0.5*sin(...)`, never on/off; add a slow global drift (offset the pattern
  coordinate by time) so in-place twinkle doesn't read as a still. Give each
  motion its own `0`-means-off slider; keep lighting in screen space.
- **Fewer + larger beats many + smaller.** Dense small elements at 1080p turn
  into visual static. A first pass is almost always too dense.
- **Palette:** deep + tint for the gradient, one or two element colors, one
  accent that behaves as a light source. Linear `exp(-d*k)` falloff for broad
  ambient light, gaussian `exp(-d*d*k)` for compact blooms. Keep bloom tight or
  the darks go to grey haze.
- **Light grounds are a different construction:** marks are subtractive
  (`col *= mix(vec3(1.0), ink, coverage)`), no tonemap, grain only, lift the
  center toward paper instead of vignetting.
- **Speed = 0 should still be a good background**, and the DEFAULTS are the
  deliverable: the instant-drop look must be the calm, professional one.
- **Don't converge on one look.** Gather evidence (palette, shape language,
  motion personality) first, then offer 2–3 genuinely different directions.

Deep detail, with the calibrated numbers and code idioms:
- [references/design-guide.md](references/design-guide.md): layered recipe,
  composition, motion, depth, hashed element fields, connected grid structures
  (circuits, mazes), density, palette and falloff.
- [references/design-light-finishing.md](references/design-light-finishing.md):
  light grounds, the cut-paper planes recipe, the finishing pass, transparency.
- [references/genres-and-references.md](references/genres-and-references.md):
  panel/property design, the genre → technique cookbook, anti-convergence,
  matching a reference video, approval snapshots and default discipline.
- [references/starter.tscshadervid](references/starter.tscshadervid): a minimal
  valid skeleton wired to the four-layer stack (6 properties, full finishing
  pass). It is an **asset**; ask the user before fetching it. Copy-to-start and
  replace the placeholder blobs with a real motif; never ship them.

## Authoring workflow

1. **If matching a reference** (video/image), read
   `references/genres-and-references.md` first: sample frames across the whole
   duration, find the loop period, isolate the background, identify the
   *procedural structure* rather than literal pixels.
2. **Pick the properties** and confirm them with the user before building.
3. **Write the shader** with clean sections and named uniforms. Keep it lean:
   strip dead code and unused variables.
4. **Run the parity checklist.**
5. **QA in CamRust** (next section). Iterate one targeted change at a time.
6. **Test extremes**: min and max of every slider must still look intentional.
7. **Snapshot on approval**: copy the approved file under a new filename and a
   distinct `"name"` before experimenting further.
8. **After the LAST edit, re-render the pure defaults** and compare to the
   reference again before declaring done. A new property must be
   default-invariant (e.g. `radius*0.055*_Thickness` with `Thickness = 1.0`):
   prove it by pixel-diffing a default render against the last approved frame.

## QA loop in CamRust (required)

Work in a scratch project so QA imports don't litter the user's Media Bin:
`project_new` (e.g. a temp `.cmproj`, 1920x1080), and `project_close` it when done.

1. **Import**: `import_media` the shader. Any `warning` = stop and fix
   (compile error, or a `vertexCode` the renderer doesn't draw). Check the
   returned `properties` match your effectDef names.
2. **Place**: `add_media_to_timeline` with the `source_id`, `start_s: 0`,
   `duration_s` long enough for the times you'll sample (e.g. 30).
3. **Render**: `render_frame` at a few times (e.g. 0, 3, 12, 25 s) with
   `inline: true` to see each frame; the PNG path is returned too.
   - All frames identical → motion isn't wired to `_Time` (or `Speed` default is 0).
   - Black or empty frame with no import warning → check the pitfalls below
     (`half`, negative `pow`, stray quote), then property-name parity.
   - Blown-out or banded → tonemap, grain, `clamp`.
4. **Compare to the reference**: view render and reference side by side, then
   numerically: downscale the render to the reference size, blur both, diff a
   grid of sample points (convert both images to RGB first). The numeric pass
   catches color drift that eyeballing misses.
5. **Non-default values**: read the clip with `get_property` (no path) to find
   its parameters at `/sourceEffect/parameters/<Name>` (a Color is
   `<Name>-red/-green/-blue/-alpha`), then `set_property` on that path and re-render. Or put the
   value in as `defaultValue` in a throwaway copy and import that.
6. **Edits**: re-import under a fresh versioned filename
   (`myshader_v3.tscshadervid`) and swap it in with `replace_media`, or place
   it fresh. Re-render the SAME times each round so cause and effect stay obvious.
7. **Transparency**: a single shader over the empty canvas can't prove alpha.
   Import a solid bright image, put it on the bottom track and the shader above,
   then render: backdrop showing through = alpha works; gaps rendering WHITE =
   you forgot to premultiply; gaps in the shader's own base color = alpha isn't
   reaching the output. `export` to `.webm` also keeps transparency.

## GLSL pitfalls (they bite in both renderers)

- **Never name a variable `half`, `float2`, `float3` or `float4`.** On macOS
  Camtasia translates GLSL to Metal, where these are real types: the shader
  compiles clean and renders nothing. Grep for them first when frames are empty.
- **Never `pow(x, 2.0)` when `x` can be negative**: `pow` is undefined for a
  negative base and gives NaN/black speckles. Use `x*x`. Keep `pow` for
  non-negative bases (tonemap, falloff on a length).
- **No `"` in `code`** (checklist item 4). A bisect that strips comments hides
  this bug, since the comment was the bug.
- A float uniform as a loop bound needs `int(...)`:
  `for (int i = int(_Count)-1; i >= 0; i--)`. Prefer literal `int` loop bounds:
  then every slider range is crash-safe by construction.
- **Antialias without `fwidth`**: derive AA from pixel pitch,
  `float aa = 1.5 * cellFreq / _ImageSize.y;` then `smoothstep(r+aa, r-aa, d)`.
- **Premultiply** when alpha < 1: `_FragColor = vec4(clamp(col,0.,1.)*alpha, alpha);`.
  Straight alpha makes transparent gaps add onto the layer below and blow to white.
- **Keep it lean**: accumulated complexity and dead code (unused accumulators,
  placeholder `exp()` lines) were the observed trigger of Camtasia compile
  crashes. To bisect, start from a flat color that references every uniform
  (`col += _Foo.rgb*0.0`), then add one construct at a time.

Constructs verified safe in Camtasia's compiler: `for` loops with `int`
counters (nested too); `if` inside loops; `mat2`, `mat2x2`, `mat3`; `exp2`,
`exp`, `pow`, `sqrt`, `acos`, `sin/cos`, `fract`, `floor`, `mod`, `clamp`,
`smoothstep`, `mix`, `sign`, `abs`, `length`, `dot`, `cross`, `normalize`;
ternary `?:` returning vec3; `vec2 *= mat2`; hashing via `fract(sin(...)*k)`.
CamRust compiles with a different front end (naga), so a clean CamRust import
is good evidence but stay inside this list for files bound for Camtasia too.

## Example panels

- Neon triangle tunnel (10 properties): `ColorNear`, `ColorFar`,
  `BackgroundColor`, `Speed`, `Twist`, `Wander`, `Density`, `Thickness`, `Glow`,
  `Stars`.
- Cut paper, the strongest light background of the set (11): `Paper`,
  `Tint0`–`Tint2`, `Shadow`, `Speed`, `Elevation`, `ShadowStrength`, `Scale`,
  `LightAngle`, `CenterCalm`. Recipe in `references/design-light-finishing.md`.
