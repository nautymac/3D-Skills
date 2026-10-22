---
name: skybox-stereo-depth
description: Render sky/skybox/at-infinity elements (stars, sun, moon, distant lens flares, fog plane, far skybox geometry) at the correct stereo depth in a per-eye 3D engine that uses the CLIP-SPACE separation parameterization (the `dynamic3d` form, `x += separation * (w - convergence)`). Covers why the depth → ∞ limit of the per-eye pixel shift is simply ±separation × viewWidth/2 (no FoV, convergence, aspect or stage-scale term survives), why convergence therefore never moves the sky and zoom never deepens it unless you opt in, per-stage modelview scaling (`bgScale`) and why it matters only at finite depth, the two strategies that remain for sky elements (project through the per-eye matrix vs. the analytic constant) and which to use for suns, stars, lens-flare chains and finite-depth lights, the HUD "no target" fallback at the infinity sentinel, common pitfalls (sub-pixel jitter, off-viewport coordinate wrap, double-shift through HUD paths, a "very large" depth that isn't, legacy helpers that still carry a FoV term, the two sign conventions), calibration in screen-width fractions including the divergence ceiling and the autostereo comfort range, and the exact one-shot conversion from a legacy eye-separation-in-world-units calibration. Applies to any stereo renderer that builds per-eye off-axis frustums; examples in pseudo-C, not tied to any engine.
---

# Skybox stereo depth — getting "at infinity" right

This skill is about a deceptively narrow problem: **what per-eye shift should I apply to a screen-space sprite (a star, a sun, a lens flare) so it reads as "infinitely far behind the screen" instead of pinning to the screen plane?**

The answer under the clip-space separation parameterization is one line. The traps come from everything *around* that line: helpers most engines already have for finite-depth HUD elements (crosshairs, lock-on boxes), sentinel depths that are not as far as they look, fixed-width coordinate fields that wrap, and legacy code paths that still carry a field-of-view term from an older parameterization. Getting it wrong produces stars that:

- Pin to the screen plane (no per-eye disparity at all) — the shift collapsed to zero.
- Deepen or flatten when the user zooms — a FoV term crept back in.
- Snap toward the camera when the user moves the convergence slider — a convergence term crept back in.
- Look right at one stage and broken at another — a per-stage modelview scale leaked into the depth.
- Smear a horizontal line across the screen — a shifted coordinate wrapped in a fixed-width field.

**Prerequisite:** your renderer already parameterizes stereo in clip space as described in `dynamic3d` §1 — the projection shear is `dir × separation`, the eye baseline is derived per frame, and `separation` is the total background disparity as a fraction of screen width. If you are still on an eye-separation-in-world-units calibration, §11 has the exact conversion; do that first, because every formula below assumes the clip-space knob.

**Public reference implementation:** [perfect_dark_3D](https://github.com/oneup03/perfect_dark_3D) (`port` branch). [`port/src/stereo.c`](https://github.com/oneup03/perfect_dark_3D/blob/port/port/src/stereo.c) holds `stereoHudParallaxPx` (the finite-depth helper with the `bgScale` correction; passing the infinity sentinel gives the sky shift), `stereoHudShiftPx` (HUD depth as a fraction of the sky's disparity), `STEREO_MAX_PARALLAX_DEPTH` and the one-shot `stereoMigrateLegacyIPD`; [`src/game/stars.c`](https://github.com/oneup03/perfect_dark_3D/blob/port/src/game/stars.c) applies the per-eye star shift exactly as §7 describes.

---

## 1. The per-eye pixel shift, and what it is made of

`dynamic3d` §1.2 gives the per-eye screen-space shift a world point at forward depth `z` receives, for anything that does not go through the projection matrix:

```c
// eye_w   = per-eye viewport width in pixels
// z       = camera-forward distance to the element, same units as conv
// conv    = convergence distance (where the screen plane sits)
// dir     = per-eye sign — see §9 pitfall 7; do NOT assume it matches the shear's
float shift_px(float z, float separation, float conv, float eye_w, float dir) {
    float ratio = (z > 0.0f) ? (conv / z) : 0.0f;   // invalid / at-infinity -> 0
    return dir * separation * (ratio - 1.0f) * (eye_w * 0.5f);
}
```

Where it comes from, in one paragraph: the per-eye projection adds `dir × separation` to the `[2][0]` shear slot, which after the perspective divide offsets NDC.x by a constant `dir × separation` for *every* vertex regardless of depth. The matching view-space eye translation (`separation × tan_half_h × conv` per eye) contributes `−dir × separation × conv/z` — a term that exactly cancels the shear at `z = conv` and fades to nothing as `z` grows. Sum the two, convert NDC to pixels with `× eye_w/2`, and you have the function above. `tan_half_h` appears in the translation and is divided straight back out by the projection's own `1/tan_half_h`, which is why no FoV term survives.

**Every hand-computed per-eye shift is a special case of the projection.** Anything that goes through the actual per-eye matrix (world geometry, a sun with a real position) gets this disparity for free. The manual formula exists for screen-space sprites the projection never sees: stars rendered as fill-rects from direction vectors, HUD-plane crosshairs, lens-flare chains anchored to a pre-projected screen X.

---

## 2. The depth → ∞ limit — what "skybox depth" actually is

Take `z → ∞`. `conv / z → 0`, and the formula collapses:

```
shift_∞ = −dir × separation × eye_w / 2
```

That is the **skybox-depth shift**: the upper bound on uncrossed disparity anything can have in this projection. Every "very far" object is pushed toward it.

### What survives, what doesn't

| Term | At infinity |
|---|---|
| `dir` | ✅ stays — disparity has a sign per eye |
| `separation` | ✅ stays — it *is* the sky's disparity |
| `eye_w` | ✅ stays — pixel units |
| `z` | ❌ gone (collapsed) |
| `conv` | ❌ gone — convergence only moves what sits in front of the screen |
| FoV / `tan_half` | ❌ never there — cancelled in the derivation |
| aspect | ❌ never there |
| per-stage `bgScale` (§5) | ❌ gone — it multiplies `z`, and `1/(bgScale × ∞)` is still zero |

### Sanity check

With `separation = 0.04` and a 320-pixel N64 eye:

```
shift_∞ = 0.04 × 320 / 2 = 6.4 px per eye, 12.8 px between the eyes
```

which is, by definition, 4% of the screen width. `separation` reads directly as the sky's on-screen disparity — that is the entire reason to prefer this parameterization, and it is what the calibration section (§10) leans on.

---

## 3. Convergence does not move the sky

Convergence picks where the screen plane sits in world space. Objects at exactly `z = conv` have zero disparity; nearer ones pop out; farther ones recede — up to the `shift_∞` ceiling, which convergence does not touch.

So the user-facing question *"when I move the Convergence slider, should the sky move?"* is answered by construction: **no.** Raising convergence pushes the screen plane out and brings the near field back toward it; the sky stays where it was. There is no "sky decoupled from convergence" mode to build and no magic constant to fit — the old design debate existed only because the world-units form coupled background disparity to `1/conv` (§11).

Expose `separation` as the single "how deep is the sky / the scene" knob and `convergence` as the single "where is the screen plane" knob, and document them that way. If a returning user says "raising convergence used to flatten everything," that is the one behaviour that genuinely changed in the migration (`dynamic3d` §2.4), not a bug.

---

## 4. FoV: there is nothing to compensate

Under this parameterization `separation` is a clip-space quantity and is FoV-independent by construction (`dynamic3d` §3). A sniper zoom from 60° to 30° magnifies the world on screen and leaves its disparity in pixels exactly where it was. The sky, the far mountains and the crosshair all agree, on the first frame, with no easing.

**This is a choice, not a law.** Physically a telephoto lens *does* magnify disparity, so if you want zoom to deepen the sky you now opt *in*, deliberately, by scaling `separation` by something FoV-derived — for the sky and the world together, never for one alone. For games, not doing that is almost always right: a scope zoom that also triples disparity is a fusion problem, not a feature.

**The trap that remains** is a *legacy* helper. Engines that migrated from the world-units form often still have a HUD parallax helper carrying `fovScale = tan(fovy/2) / tan(default_fovy/2)` and a `cot(fovy/2)` projection term. Those cancelled each other at infinity in the old form, which is why nobody noticed them. Under clip space there is nothing for them to cancel against, and a sky computed through such a helper deepens under zoom while the world does not. Read every helper you reuse; if it mentions FoV at all, it predates the migration and should be reduced to §1's `shift_px`.

---

## 5. Per-stage modelview scaling (`bgScale`)

Some engines — N64 ports in particular — apply a per-stage scale to their modelview matrices (often 0.5 or 2.0, for fixed-point precision). A vertex at world depth `z` reaches the projection at `z × bgScale`, so the manual formula must use the same effective depth or it disagrees with what the matrix does:

```c
float effective_z = z * bgScale;
shift = shift_px(effective_z, separation, conv, eye_w, dir);
```

**This matters at finite depth** — without it the crosshair drifts off the laser dot on Perfect Dark's Villa level (`bgScale = 0.5`) when zoomed into the scope.

**It does not matter at infinity.** `conv / (bgScale × ∞)` is zero for any `bgScale`. Sky elements that go through the limit can ignore it; the finite-depth helper should apply it unconditionally so the same function is correct for both.

If your engine has no per-stage modelview scale, ignore this section.

---

## 6. Two strategies for sky elements

The world-units form needed four strategies, because reusing a HUD helper at a huge depth silently changed the FoV behaviour and decoupling from convergence needed its own helper. Under clip space those distinctions collapse: reusing the finite-depth helper at the infinity sentinel *is* the analytic limit, and decoupling from convergence is the default. Two remain:

| Strategy | How it works | Use when |
|---|---|---|
| **A. Project through the per-eye matrix** | Build the element as a 3D vertex at a large world Z, push it through the engine's per-eye projection, read off the screen position per eye | The element has a sensible world position (sun, moon, world-placed lights). Convergence, `bgScale` and any deliberate FoV scaling all just work, because the matrix is the source of truth. |
| **B. Analytic infinity constant** | `shift_∞ = −dir × separation × eye_w/2`, computed once per eye per frame and added to the mono screen X — or, equivalently, `shift_px(INFINITY_SENTINEL, …)` | The element is direction-only (stars on a celestial sphere, fill-rect sprites) with no real world Z. Cheapest possible; exact. |

When you can't touch the sky code at all (a Vulkan injector rewriting SPIR-V), Strategy B can still be applied per vertex in clip space — `x -= dir·separation·w`, plus `z = w` to pin it to the far plane — see `vulkan-multiview-stereo` §6.2, including the stretch-to-cover for fullscreen sky quads and the reversed-Z caveat.

### Recommendation by element

| Element | Strategy | Why |
|---|---|---|
| **Sun / moon orb** | A | It has a world position; let the matrix do it. The lens flare anchors to the result. |
| **Stars** (fill-rects from precomputed direction vectors) | B | No world Z. One constant, applied per visible star. |
| **Far-world geometry** (skybox quads, distant terrain, fog plane) | A — already happens | Ordinary world geometry through the per-eye matrix. Nothing to do. Tag the projection as "world" if your renderer classifies projections (see `rt64-stereo-3d` step 6 — the skybox gets its own transform ID precisely so it receives the world treatment). |
| **Lens flare chain** (sprites from a light toward screen centre) | Inherit the anchor's | Anchor at the per-eye-shifted light X and extend the chain from there. Do not shift each chain element separately — they share the anchor's disparity. |
| **Lights at finite depth** (a lamp's glare, a torch) | `shift_px` at the light's real depth × `bgScale` | Not sky. Full formula, finite `z`. |
| **HUD / crosshair with no target** | `shift_px(INFINITY_SENTINEL, …)` | The reticle should recede to the sky rather than snap to the HUD plane (`dynamic3d` §5.1). Same function, same sentinel. |

---

## 7. Per-element guidance

### Sun / moon orb (Strategy A)

```c
vec3 sun_world = { ... };                       // far away, real world position
vec3 sun_cam   = view_matrix[current_eye] * sun_world;   // per-eye view (eye translation applied)
vec4 sun_clip  = proj_matrix[current_eye] * sun_cam;     // per-eye projection (shear applied)
float ndc_x    = sun_clip.x / sun_clip.w;
int   screen_x = (int)floorf((ndc_x + 1.0f) * 0.5f * eye_w + view_left + 0.5f);
```

Per-eye `screen_x` differs by the right amount for all the right reasons. For a flare chain, anchor at the per-eye `screen_x` and compute chain positions as offsets from the anchor toward screen centre; the chain inherits the sun's disparity, which is what viewers perceive as correct.

### Stars (Strategy B)

Stars are usually unit vectors on a celestial sphere projected with a rotation-only matrix. They have no world Z, so compute the constant once per eye per pass:

```c
const float star_shift_px = shift_px(1.0e9f /* infinity sentinel */,
                                     separation, conv, eye_w, dir_for(current_eye));
// == -dir * separation * eye_w / 2 to float precision

for (each visible star) {
    float sx = engine_star_screen_x(star);             // engine's normal math, float
    float sy = engine_star_screen_y(star);
    int draw_x = (int)floorf(sx + star_shift_px + 0.5f);   // round — §9 pitfall 1
    int draw_y = (int)floorf(sy + 0.5f);
    if (draw_x < view_left || draw_x >= view_right) continue; // clip — §9 pitfall 2
    if (draw_y < view_top  || draw_y >= view_bottom) continue;
    emit_fillrect(draw_x, draw_y);
}
```

Two adds and a round per star. This is exactly what perfect_dark_3D's `stars.c` does through `stereoHudParallaxPx(eye, 1.0e9f, viewwidth)`.

### Lens flares anchored to finite-depth lights

```c
vec3  light_cam = view_matrix * light_world;
float z         = -light_cam.z;                      // positive forward distance
float shift     = shift_px(z * bgScale, separation, conv, eye_w, dir);
glare_screen_x += shift;
```

### HUD depth as a fraction of the sky (the tidy way to expose a HUD slider)

Because the sky's disparity is `separation × eye_w/2`, a HUD-depth slider expressed as a **fraction of that** has a natural, bounded meaning:

```c
// hud_depth in [-1, +1]: +1 = as far as the sky, 0 = screen plane,
// -1 = the same magnitude of crossed (pop-out) disparity.
float hud_shift_px = dir * separation * hud_depth * eye_w * 0.5f;
```

The HUD can never diverge further than the sky does, and `separation = 0` flattens the HUD along with everything else. Convergence cancels here too (`dynamic3d` §5.2), which is why this mapping is worth ruling out early when debugging convergence-dependent symptoms.

### Distant world geometry

Skybox quads, far-clip meshes, fog planes — anything already going through the per-eye matrix needs **no special code**.

---

## 8. Where to add the shift

Add the per-eye pixel shift to the **screen-X** the engine's mono path already computed, immediately before emitting. Don't pre-translate world coordinates for a direction-only sprite, and don't post-compose with another matrix:

```c
float sx = engine_project_to_screen_x(element);   // mono path, unchanged
if (stereo_active) sx += per_eye_shift_px;         // stereo: one add
draw_sprite_at(round(sx), sy);
```

The simpler the integration point, the easier the sign test in §9 pitfall 7 is to run.

---

## 9. Pitfalls

1. **Sub-pixel jitter on round-down.** Casting a float screen position to `int` truncates, so as the camera rotates through sub-pixel positions the integer X flickers by one pixel; the per-eye shift can land the two eyes on different sides of an integer boundary and make the flicker stereo-visible. Round (`floorf(x + 0.5f)`), don't truncate.

2. **Off-viewport coordinate wrap.** Fill-rect commands with fixed-width coordinate fields (N64 GBI: 10-bit X, modulo 1024; some D3D paths: signed 16-bit) produce garbage when a coordinate leaves the representable range. A star whose shifted X lands at −1 wraps to ~1023 and emits a rectangle smeared across the whole screen. Clip explicitly per eye before emitting; a star that is visible in one eye and one pixel off-screen in the other is invisible to the viewer.

3. **Double shift through the HUD path.** If the sky element draws through a path that *also* applies a per-eye shift (a rasterizer that auto-shifts when Z-test is off or an "aspect/letterbox" flag is set, an overlay pass that applies `hud_shift_px`), and you add a manual shift too, it moves twice. Either opt the sky draw out of the automatic shift (re-enable Z / clear the flag around it) or drop your manual shift and rely on the automatic one — not both.

4. **"Very large" is not infinity.** The finite-depth formula is `conv/z − 1`. At `z = 1000, conv = 100` the shift is 10% short of the limit; at `z = 1e9` the shortfall is `1e-7`. Pick a sentinel of **`1e6` or larger**, small enough that `float` arithmetic in the formula stays clean (`1e9` is fine; `FLT_MAX` is not).

5. **Legacy helpers that still carry a FoV term.** §4. If the "HUD parallax" helper you are about to reuse mentions `fovScale`, `cot(fovy/2)`, `P00` or a default/reference FoV, it predates the migration. Under clip space it is wrong at every depth, not just at infinity. Reduce it to §1's `shift_px`.

6. **Direction-only stars faked as world points.** Multiplying a unit vector by a huge radius and projecting through the matrix (Strategy A for stars) mostly works, but float precision in the projection's intermediates can produce coordinate jitter the original direction-only projection never had. Strategy B avoids it. If you insist on A, use a radius large enough that `conv/radius` is negligible and small enough that intermediates don't snap — `1e6 × conv` is reasonable.

7. **Two sign conventions, not one.** The projection shear's `dir` and the screen-space compositor's `dir` are independent and, in at least one verified implementation, opposite (`dynamic3d` §1.4). A sky sprite shifted with the shear's sign while the world uses the compositor's — or vice versa — sits *in front* of the screen, with crossed disparity, and reads as "the stars are floating in the room." Test: with the sky shift enabled, the star field must show **uncrossed** disparity (left-eye image displaced *left* of the right-eye image), the same direction as the far mountains that go through the matrix. Test it once per code path; don't infer it from the other path.

8. **Convergence used as a depth in a threshold.** If a "is this element far enough to count as sky" gate divides by `conv`, it retunes itself whenever the user moves a comfort slider (`dynamic3d` §6.2). Compare against a fixed world depth, or against the sentinel.

---

## 10. Calibration

### The knob is the calibration

`separation` is the sky's disparity as a fraction of screen width, so there is no world-units constant to fit and no "solve for IPD given a target percentage" step. Pick the target directly:

| Total background disparity (`separation`) | Reads as |
|---|---|
| below ~0.01 | too subtle to register as 3D |
| **0.04 – 0.05** | comfortable default on glasses and autostereo panels; perfect_dark_3D ships 0.04 |
| ~0.08 | aggressive; fine on large glasses-based displays, marginal on autostereo |
| `IPD / screen_width` ≈ **0.105** on a 27-inch 16:9 monitor | the divergence ceiling — uncrossed disparity beyond the viewer's eye spacing cannot be fused at any comfort level |

Autostereo (lenticular) panels ghost more than glasses and their viewers fatigue sooner; treat roughly 0.04–0.05 as the practical ceiling there and 0.08 as the ceiling for glasses. Because the value is display-relative rather than world-relative, it is directly comparable across games — the same `0.05` means the same thing in Banjo-Kazooie and Perfect Dark despite their world units differing by orders of magnitude.

### Slider defaults and ranges

| Slider | Range | Default | What it does |
|---|---|---|---|
| `Stereo.Separation` | 0 – 0.15 | 0.04 – 0.05 | Total background disparity, fraction of screen width. Scales all stereo linearly. |
| `Stereo.Convergence` | `[1.5 × near, 0.9 × far]` (engine-clamped) | mid-scene, e.g. typical aim distance | Where the screen plane sits. Does not affect the sky. |
| `Stereo.HudDepth` | −1 … +1 | 0 – 0.5 | HUD plane as a fraction of the sky's disparity (§7). |

The convergence clamp is now a *comfort* limit, not a numerical one: under clip space nothing explodes as `conv → 0` (the shear is just `separation`), but the near field pops out unboundedly. Keep the clamp, fix its comment.

### "It looks wrong" diagnostics

1. **Sky has no depth at all (no per-eye difference)?** `stereo_active` false when the shift is computed; `separation == 0`; `dir` read from a stale eye rather than the current one; or the shift added before a step that recomputes X.
2. **Sky deeper than the far world (mountains sit in front of the stars)?** A legacy constant or helper is inflating the sky (pitfall 5), or the world projection is not receiving the stereo treatment (untagged skybox projection in a renderer that classifies projections).
3. **Sky moves when I move Convergence?** A `1/conv` term survived somewhere — a legacy helper, or a hand-derived formula. `shift_∞` has no convergence term; anything that produces one is wrong.
4. **Sky deepens or flattens when I zoom?** A FoV term survived (pitfall 5). Delete it; if you want zoom to deepen the scene, scale `separation` for everything, deliberately.
5. **One stage's sky is off, others fine?** `bgScale` at infinity cannot do this (§5). If Strategy A is in use, that stage's view matrix has pushed your "sky sentinel" point to a finite post-scale depth — raise the sentinel Z.
6. **Stars float in front of the screen?** Sign convention (pitfall 7). Run the uncrossed-disparity test.

---

## 11. Coming from an eye-separation-in-world-units calibration

You only need this section if the renderer — or a saved user config — still expresses stereo as a physical eye separation (`IPD`) in world units. The old per-eye shift was

```
shift_px = −dir × IPD × eye_w × (1/z − 1/conv) / (4 × tan(fovy/2) × aspect)
```

and its infinity limit `dir × IPD × eye_w / (4 × conv × tan(fovy/2) × aspect)` is where the old design's convergence coupling, FoV-compensation trap and "solve for IPD" calibration all came from. Divide that limit by `eye_w/2` and you have the clip-space knob:

```c
separation = IPD / (2.0f * conv * tanf(radians(reference_fovy) * 0.5f) * aspect);
```

This is **exact**: at the user's saved convergence and the reference FoV it reproduces their previous image pixel-for-pixel. Run it once on config load, gated on the new key being absent while the old one is present, log it, and retire the old key — `stereoMigrateLegacyIPD` in perfect_dark_3D's `stereo.c` is a worked example (reference FoV 60°, reference aspect 16:9, result clamped to `[0, 0.15]`). The full per-site migration checklist, including what to delete (the reference-FoV slider, the FoV EMA, any world-scale multiplier), is `dynamic3d` §2.

After migrating, the one visible change is the one in §3: convergence no longer moves the sky. Say so in the release notes.

---

## 12. Quick reference

```c
// --- the one formula (dynamic3d §1.2) -------------------------------------------
shift_px(z)  = dir * separation * (conv/z - 1) * eye_w/2;      // finite depth
shift_inf    = -dir * separation * eye_w/2;                     // z -> infinity
// separation == total background disparity as a fraction of screen width.

// --- what the sky shift depends on --------------------------------------------
//   dir, separation, eye_w.      NOT: z, conv, FoV, aspect, bgScale.

// --- strategies ------------------------------------------------------------------
//   has a world position  -> project through the per-eye matrix (A)
//   direction-only sprite -> shift_inf, or shift_px(1e9, ...) (B); same number
//   finite-depth light    -> shift_px(z * bgScale, ...)
//   HUD, no target        -> shift_px(1e9, ...)
//   HUD at a chosen depth -> dir * separation * hud_depth * eye_w/2, hud_depth in [-1, 1]

// --- calibration -----------------------------------------------------------------
//   separation default 0.04-0.05; autostereo ceiling ~0.05; glasses ~0.08;
//   divergence ceiling IPD/screen_width ~ 0.105 on a 27in 16:9.

// --- migration from world-units IPD (once, on load) ----------------------------
//   separation = IPD / (2 * conv * tan(ref_fovy/2) * aspect);   // exact
```

---

## 13. What this skill does NOT cover

- **Building the per-eye projection and the clip-space parameterization itself** — `dynamic3d` §1 (the shear, the derived eye baseline, the sign-convention trap).
- **Composing the two eye images for display** (SbS / TaB / interlaced / anaglyph / LeiaSR weave) — `output3d` and `extraoutput3d`.
- **Auto-convergence** — it uses the same `shift_px`, but the depth-buffer-driven control loop is `dynamic3d` §4.
- **Depth-aware crosshairs and the fit-vs-cover rule for shifted overlays** — `dynamic3d` §5.
- **Which projections an engine treats as "world" vs "HUD"** and how a skybox projection gets tagged — engine-specific; `rt64-stereo-3d` step 6 for RT64 ports.
- **Head tracking** — the formula assumes a fixed viewer; head-coupled perspective offsets the eye positions before any of this runs.
- **VR headsets** — per-eye full-frame textures with lens distortion are a different geometry. This skill is about shared-screen stereo (one panel: glasses, lenticular, or anaglyph).
