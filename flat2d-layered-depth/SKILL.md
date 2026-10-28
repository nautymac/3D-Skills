---
name: flat2d-layered-depth
description: Give stereo 3D depth to FLAT 2D games — pixel art, hand-drawn, comic, Flash-like — whose draws carry no depth at all (orthographic, clip w == 1, so the usual `x += separation * (w - convergence)` yields zero disparity), by inferring depth from HOW the game draws. Covers sorting games into kinds (2.5D/3D → ordinary stereo instead; layer priority in z; separately drawn sprites → this skill; one finished picture → only AI 2D-to-3D), a generated geometry shader behind the game's vertex shader that sees whole triangles and shifts each sprite per eye without CPU read-back (renaming vertex outputs, per-quad not per-triangle, interface blocks, split vertex stages), the two depth rules (feet: floor planes slope with screen height and sprites stand at their bottom edge — adventures/AGS; order: consecutive draws form layer groups whose depth follows their position in the composed frame — side-scrollers, top-down, Unity 2D), keeping UI at the screen (scissor/viewport batches), following the composition instead of the raw draw order (low-resolution layer caches promoted to stereo, composite layout expansion, texture views and copies, sampled units only), the exclusions that otherwise wreck it (colour-grading LUT strips, scratch/glyph targets, stencil-only clears, repeated panels, walk-behinds, Unity 2D lit sprites and full-screen post passes), fixed-function and ARB-assembly games, measuring with eye dumps and dense disparity maps instead of eyeballing, a symptom→cause→fix table, and per-engine starting settings (AGS, Unity 2D, Chowdren/Clickteam, LÖVE, PSSG/Cg, Flash-like fixed function, Adobe AIR, custom engines). Use it when a 2D game is flat in stereo, when only its HUD has depth, or when layers come out at the wrong depth.
---

# flat2d-layered-depth — stereo depth for 2D games that send none

> *"The wrapper is in, both eyes render, but the game is a flat sheet. It's a 2D game."*

Geometric stereo shifts each vertex by `separation * (w - convergence)`. A 2D game draws everything orthographically: every vertex has `w == 1`, so every pixel gets the same shift, and the picture is a flat sheet at one depth. To give it depth you have to **invent** a depth for each sprite from the structure of the frame: where the sprite sits, how big it is, which layer it belongs to, and which render target it ends up in. This skill is the playbook that made about fifteen such games work, among them adventures, side-scrollers, top-down shooters, a comic and Flash-like point-and-clicks.

*Added in the nautymac fork. Distilled from [OGL3D-SR](https://github.com/nautymac/OGL3D-SR) (`flato3d` branch: `src/inject.cpp` builds the geometry shader, `src/stereo.cpp` holds the rules) and [Flato3D-SR](https://github.com/nautymac/Flato3D-SR) (profiles, measurements, `docs/WORKFLOW.md`). Verified with SR weaving and side-by-side output.*

Related skills: `wrapper-injection` (getting an OpenGL wrapper into the game, including D3D games through WineD3D), `output3d` (composing the eyes), `dynamic3d` (auto-convergence, which you turn **off** for flat depth).

---

## 1. Sort the game first

| Kind | What reaches the GPU | What to do |
|---|---|---|
| 1. 2.5D / 3D | Perspective cameras (`w != 1`) | Ordinary geometric stereo with auto-convergence. If you apply flat depth to it, the whole scene sticks to the far end as one sheet. (Snufkin looks 2D but has a perspective camera.) |
| 2. Layer priority in z | Orthographic, but z encodes the layer (DS-style engines) | Drive the shift from clip z |
| 3. Separately drawn sprites | Many draws, each sprite or layer its own quad(s) | **This skill** |
| 4. One finished picture | A few draws per frame; the scene was composed on the CPU (Blade Runner: Enhanced Edition draws 6 per frame), or the content is video (FRAMED's comic panels are WebM) | No layers exist. Use AI 2D→3D; a silhouette trick can lift dark figures (§7) |

Check by tracing one frame: count the draws and look at what each draws into.

## 2. Where the depth is applied: a generated geometry shader

You need each sprite's screen rectangle to decide its depth. A vertex shader sees one vertex; reading positions back on the CPU stalls. Insert a **geometry shader** behind the game's vertex shader:

1. Rename every output of the game's vertex shader (`foo` → `s3d_v_foo`). GLSL reserves names with `__`, so an output that starts with `_` needs another prefix (LÖVE's shaders do this).
2. Generate a GS that takes `triangles`, computes the triangle's bounding box in NDC, decides a per-vertex shift, and re-emits the vertices with the outputs **under their original names**. The fragment shader then links unchanged.
3. Pass array outputs (`ffp_varying_texcoord[8]`, which needs `GL_ARB_arrays_of_arrays` below GLSL 4.30), interface blocks, and the legacy built-ins (`gl_FrontColor`, `gl_TexCoord` …) in compatibility contexts. Look for them inside macros too: MojoShader (FNA) writes them only through `#define vs_oT0 gl_TexCoord[0]`. A scan that skips preprocessor lines passes nothing on, and the screen goes black (with stereo off too). Count the bodies of the macros the code uses, not every macro, or a core-profile shader turns into a compatibility one.
4. Only touch triangles with `w == 1` on all three vertices. Perspective geometry keeps the normal stereo path.
5. Shift **per quad, not per triangle**: `floor(gl_PrimitiveIDIn / 2)`. If the two triangles of a full-screen quad get different depths, a diagonal line runs across the screen.

```glsl
// per draw: s3d_P = (eye -1/+1, separation, …); s3d_F = (rule, depth at top, plane width, depth at screen plane)
// s3d_D = (screen-fixed / lit-sprite flag, rule params…)
void main() {
    vec4 p0 = gl_in[0].gl_Position, p1 = gl_in[1].gl_Position, p2 = gl_in[2].gl_Position;
    vec3 sh = vec3(0.0);
    if (s3d_P.x != 0.0 && s3d_F.x != 0.0 && abs(s3d_D.x - 1.0) > 0.5 &&
        abs(p0.w - 1.0) < 1e-5 && abs(p1.w - 1.0) < 1e-5 && abs(p2.w - 1.0) < 1e-5) {
        vec2 lo = min(min(p0.xy, p1.xy), p2.xy), hi = max(max(p0.xy, p1.xy), p2.xy);
        vec3 d = /* rule: §3 */;
        sh = s3d_P.x * s3d_P.y * d;
        if (s3d_D.x > 1.5 && all(lessThanEqual(lo, vec2(-0.99))) && all(greaterThanEqual(hi, vec2(0.99))))
            sh = vec3(0.0);                         // a lit sprite's draw covering the whole screen = post pass
    }
    for (int i = 0; i < 3; ++i) {
        gl_Position = gl_in[i].gl_Position;
        gl_Position.x += sh[i] * gl_Position.w;
        /* copy every renamed output: foo = s3d_v_foo[i]; … */
        EmitVertex();
    }
}
```

Practicalities:

- Cache the generated GS by (GL context, VS source). Some translation layers delete contexts and reuse GL names.
- A program whose vertex stage is split into several shader objects (WineD3D links one with `main()` and one without it that writes the outputs) needs the outputs of **all** pieces merged. If any piece cannot be parsed, give the program no GS rather than a broken one.
- No GLSL at all? Engines built on NVIDIA Cg use `!!ARBvp1.0` programs: rewrite the program's position output and feed the per-draw shift through `program.env`. Fixed-function games: shift the orthographic projection per draw on the CPU.
- Leave points and lines alone unless you write a matching GS for them.

## 3. The two depth rules

Depth runs from 0 (screen) to `depth_top` (farthest). `depth_screen` moves the whole range so that some depth sits on the screen plane (0.3 is a good start, so the HUD does not pop out).

### 3.1 `feet` — where a thing stands (adventures, AGS, point-and-click)

- **Planes:** quads at least `plane_size` wide **and** `plane_height` tall in NDC are floors or backdrops. Each vertex gets the depth of its own screen height (the top of the screen is far), so the floor slopes away.
- **Sprites:** everything smaller stands at the depth of its **bottom edge**, the feet. A character is as deep as the floor where it stands, and its head does not lean back.
- `sprite_min` is the nearest any sprite may come. Make it relative to `depth_screen`, or objects near the bottom sink behind the floor when the user pulls the scene forward.
- `plane_tall`: a picture drawn as several narrow vertical pieces should count as floor slope if a piece is at least this tall, whatever its width. Otherwise each piece stands at its own bottom edge, and lines show where pieces meet (Samorost 2).
- In fixed-function games the CPU can group pieces that touch top to bottom (legs, body, head) into one object at one foot height. An object drawn over an earlier object is not placed behind it. A screen-wide picture drawn **after** the sprites (an inventory bar), and everything inside it, sits at the screen (Machinarium).

### 3.2 `order` — drawing order (side-scrollers, top-down, Unity 2D)

- **Group consecutive draws into layers.**
  - `group = texture`: the group ends when the program or the texture on unit 0 changes. Grouping by program alone made one fixed-function game a single layer.
  - `group = draw`: one draw is one layer. Unity draws each parallax plane separately from the same atlas.
- **Depth from position.** Each group's depth comes from its position in the **composed** frame (§4): the first group is farthest and the last is at the screen. `order_curve < 1` (0.5) gives more of the range to the far layers when many background layers would otherwise bunch together.
- **Low-resolution targets: shift by whole pixels.** A game that draws at 432x240 and scales up shows every fraction of a pixel: with a fractional per-eye shift the two eyes round a sprite at a fractional scroll position differently, and a parallax layer's disparity flips by a pixel while it scrolls (the tiles sway). Round the shift to whole pixels of the target (half the viewport width per NDC unit) in the geometry shader, and drop the spread below, which is less than a pixel and would round differently per sprite.
- **Spread inside a group by primitive index.** A batch is drawn back to front, so its first sprite is the farthest. Spread at most ~80 % of the group's slot, per quad.
- **Keep depths stable over time.** Smooth each group's depth per identity (program + nth group of that program). A group missing from the last layout (an animated sprite that switched texture) goes right after the previous known group, not to the end. Restart smoothing after a group has been absent for 3 frames. Depth is a place among the places, so a group that comes and goes (a blinking text cursor, an object scrolling into view) moves every other layer; keep the place count at its recent peak and let it shrink only after a few seconds, or at once when it drops by more than a couple (another scene). Without that, Axiom Verge 2's layers glided back and forth while walking.

### 3.3 UI stays at the screen

Many engines change scissor or viewport between the world and the UI. Count **batches**: a scissor/viewport call starts a new batch, and draws in batch ≥ `screen_batch` are UI at screen depth (AGS: 0 is the room, 1 is overlays/GUI, 2 is the cursor).

Two traps:

- A scissor/viewport call only starts a batch if the **next draw goes to the same render target**: the framebuffer *and* its colour attachment. Engines reset the scissor for every pass, and counting those calls put a whole scene in the "UI" batch. FNA renders every target through one framebuffer object and swaps its attachment, so comparing framebuffer numbers alone did the same (disparity 0 everywhere).
- Games and translation layers that set a viewport or scissor for **every** pass (Unity through WineD3D sets it between 2D-light passes; FRAMED sets a scissor per draw) need `screen_batch = 0`, i.e. off.

## 4. Follow the composition, not the draw list

The draw order is not the order on screen. Engines draw layers into off-screen targets and composite them later. Build a per-frame **event list** (draws, clears, blits, copies, with their targets and sampled textures), then lay out the frame from the final picture backwards.

- **Root:** the target of the last draw that used a program; if there is none, the last event. Do not root at trailing blits, and never at a bloom buffer that a later rule happened to prefer.
- **Composites:** a draw that samples a stereo render target is a composite. Expand it into the layers of the target it samples, in place, so a city backdrop drawn after a cache still sits behind it on screen.
- **Layer caches:** some games draw the whole level into a low-resolution mono buffer and then stretch it to the screen (Door Kickers: ~416×234 inside a 1024² target). Promote such a framebuffer to stereo when a draw into a stereo target samples it, and only if it was drawn into this frame. Otherwise upload and clear buffers get promoted and every sprite atlas becomes a "composite". Scale the shift inside it by the cache-to-screen ratio (from its scissor/viewport, or a per-game `cache_scale`). Depth inside a low-resolution cache comes in whole cache pixels, so steps are coarse.
- **Track what aliases what.** Texture views (`glTextureView`, which is how translation layers express SRVs/RTVs) of a stereo texture need the same view of its right-eye clone. Clones need immutable storage, including `glTexStorage2DMultisample`. `glCopyImageSubData` and blits must be mirrored to the clones. Follow copy chains. Compare targets by the texture they view, not by FBO id. Key the composition order (events, sources, scratch targets) by framebuffer + colour attachment as well: under FNA's single framebuffer object every target looked like one scratch target, cleared and redrawn many times a frame, and the scene collapsed into one layer. Keep a framebuffer's first attachment under its own number so engines with one target per framebuffer are unaffected, and give later attachments numbers of their own; a sampled texture whose framebuffer has moved on is found through the target it was last drawn into as.
- **Only units the shader reads count.** Engines and translation layers leave old render targets bound. Take the sampled units from the program's sampler uniforms (or `texture[n]` in ARB fragment programs). Otherwise a leftover target makes every sprite look like a composite.

### 4.1 Exclusions that are needed in practice

| Thing | Rule | What went wrong without it |
|---|---|---|
| Colour-grading LUT (Unity post-processing, a 1024×32 strip) | A target with one side more than 4× the other is never a layer cache | The LUT became stereo, was shifted in the right eye, and turned the right eye red |
| Scratch / glyph targets | A target cleared (with colour) and drawn several times per frame gets one layer per use | Each glyph came one layer nearer, so text lines slanted |
| Stencil-only clears | Do not count as a scratch restart | The scene clumped into 2-3 layers |
| The root | Never a scratch target | The scene clumped whenever the HUD was visible |
| Repeated structures | A run of draws (same program, same counts) in another scissor region repeats the earlier run's depths, if the next group repeats too | Comic panels and normal-map passes stepped through depths |
| Walk-behinds | The same texture redrawn with the same position mapping takes the original picture's layer, unless it covers objects drawn in between | A rectangle of background floated forward |
| Unity 2D lit sprites | A draw that samples a stereo target (the light texture) **and** its own ordinary texture (≥ 64 px, not a strip) is a lit sprite: give it order depth, and keep the depth of a layer it redraws | The whole Unity 2D scene counted as composites and was flat |
| Full-screen post passes | A primitive covering the whole screen stays at the screen | Post-processing shifted the whole image |

## 5. Measure, don't eyeball

- **Dump** both eye buffers on a hotkey, side by side, before output composition. Dumps are not screen captures: when the output path is in doubt (a multisampled window), check a real capture too.
- **Disparity map:** per 24-px block, the horizontal offset that best matches the right eye to the left; print the numbers over the dimmed left eye. Positive is behind the screen. Repeating patterns and flat areas give false readings (often at the search limit).
- **Area masks** for small characters, e.g. measure only purple pixels inside a rectangle.
- **A good result** (separation 25 %, 3840-wide output, `depth_screen` 0.3): far background +30…60 px, terrain/midground +8…30, characters −10…+10, HUD 0…−20, and the layer order matches what overlaps what.
- **Trace one frame**: per draw, write target (stereo/mono), program and its GS, sampled textures (and whether they have a right-eye copy), group, slot and depth. Also log the layout line (`root fbX, scratch targets …, composites fbA<fbB`). Most bugs show up as a wrong classification in that trace.

## 6. Symptoms

| Symptom | Likely cause | Fix |
|---|---|---|
| Whole picture flat | GS not attached, or every draw classified as UI/composite | Check the trace. `screen_batch = 0` for scissor/viewport-per-pass games |
| Flat and `shaders injected 0` | ARB programs or fixed function | §2 practicalities |
| One disparity for the whole scene | Scene drawn into a low-res mono cache, then pasted | Layer-cache promotion |
| Only the UI has depth | Scissor reset per buffer counted as UI batches | Batch only within the same target |
| Background in front of nearer things | Draw order ≠ composite order | Composite layout; check views/copies if the layout is empty |
| Several backgrounds at one depth | Same-atlas layers grouped together | `group = draw`, `order_curve = 0.5` |
| Diagonal line across the screen | Triangles of a quad shifted differently | Per-quad shift |
| Text lines slant | Glyph scratch target | One layer per use |
| Layers sway while scrolling, fine when still | Fractional per-eye shift in a low-resolution target; layer count changing as objects scroll in and out | Whole-pixel shifts; keep the layer count at its recent peak (§3.2) |
| Text slants under `order`, no scratch target involved | The glyphs are one draw into the scene target (GameMaker), and the group spread by primitive index walks along the line | Prefer `feet` for such games; unsolved under `order` |
| A character far back | Its texture changed, so it became a new group at the end | Place unknown groups after the previous known one |
| Character floats in front of the floor (side-scroller) | Floor drawn first = farther | `feet` for adventures; unsolved for side-scrollers with `order` |
| Whole room in front of the character standing before it (`feet`) | The room is drawn as tall narrow pieces reaching below the screen; each stands at its off-screen bottom edge, nearer than anything on screen | `plane_tall` above the characters' height (Home: 1, half the screen) |
| Right eye has different colours | LUT strip made stereo | Strip exclusion |
| One eye misses parts (`0x8cd7`) | Right-eye view of a non-immutable clone | Immutable clones, incl. multisample |
| Unity 2D: both eyes identical | Lit sprites as composites + viewport per light pass | Lit-sprite rule + `screen_batch = 0` |
| Whole scene stuck far away, near max separation | It is a 3D game (kind 1) | Flat depth off, auto-convergence on |
| Few draws, none into the scene buffer | One finished picture (kind 4) | AI 2D→3D |
| Black screen once the wrapper is in, also with stereo off; fine without it | The generated GS passes none of the outputs on: they are built-ins written through macros (MojoShader) | Scan the bodies of the macros the shader uses (§2) |
| Whole scene at disparity 0 or in one layer; every draw in the trace on the same framebuffer number | One framebuffer object for all render targets (FNA): per-target viewports start UI batches, all targets look like one scratch target | Batches and composition keyed by framebuffer + colour attachment (§3.3, §4) |

## 7. Merged pictures: the silhouette trick

When the figures arrive already merged into one picture (video panels) and they are **dark silhouettes**, rewrite that fragment shader: sample the picture through a coordinate shifted per eye wherever the shifted or unshifted pixel is dark. The figures stand in front, and the place they left shows the picture beside them. Configure it per program hash with the varying and sampler names. Bright figures need AI depth.

## 8. Starting settings per engine

| Engine / genre | Start with | Notes |
|---|---|---|
| AGS (`acsetup.cfg` `driver=OGL`) | `rule = feet`, `screen_batch = 1` | Wide walk-behinds must stay sprites, so keep plane thresholds high |
| Unity 2D (D3D11 via WineD3D) | `rule = order`, `group = draw`, `order_curve = 0.5`, `screen_batch = 0` | Read the API from the player log's `Version:` line. CSMT on. 2D lights need the lit-sprite rule; LUT strips need the strip rule |
| Chowdren (Clickteam Fusion ports) | `order` | Scratch targets for text |
| LÖVE (OpenGL) | `order` | Outputs starting with `_` |
| PSSG / NVIDIA Cg (ARB programs) | `order`, `screen_batch = 0` | ARB rewrite; forcing Cg to GLSL turned the screen white |
| Flash-like fixed function (Machinarium) | `feet` with the fixed-function grouping | `-ogl` switch injected; bitmap fills from client arrays; walk-behind rule |
| Adobe AIR (Samorost 2) | `feet`, `plane_tall = 0.12` | Pictures drawn in tall pieces |
| Custom D3D11 (Braid, Anniversary Edition) | template `order` | Needed a loader shim (`wrapper-injection` §2.2) |
| FNA (XNA reimplementation, .NET: Axiom Verge 1 and 2) | OpenGL through `/gldevice:OpenGL` (`wrapper-injection` §2.1), `order`; `order_curve = 0.4` when HUD textures are many layers | One framebuffer object for all targets; MojoShader macros; a low-resolution game buffer (480x270) gives 4 px depth steps on a 1920-wide eye |
| GameMaker Studio 2, side-view rooms (Home) | `feet`, `plane_tall = 1` | Rooms are one draw of tall pieces reaching below the screen. `order` would slant dialogue text (all glyphs in one draw) |
| Comic panels as video (FRAMED) | `order`, repeats share a layer, `screen_batch = 0`, silhouette | Inside a panel it is kind 4 |

Known unsolved: characters floating in front of the floor in side-scrollers under `order`; coarse depth steps inside low-resolution caches; bright figures inside merged pictures.
