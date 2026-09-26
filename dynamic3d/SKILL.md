---
name: dynamic3d
description: Add the adaptive layer on top of an off-axis stereo-3D renderer — Auto-Convergence (depth-buffer-driven control loop) and a depth-aware dynamic crosshair/overlay — using the CLIP-SPACE separation parameterization (NVIDIA/3Dmigoto convention, `x += separation * (w - convergence)`) rather than an eye-separation-in-metres one. Covers why the two are algebraically identical but not equally usable, a full migration path from the metres form (exact conversion formula plus a per-site checklist), why FoV auto-scaling and the reference-FoV slider disappear entirely under clip space, the sign-convention trap of having TWO independent conventions in one codebase, robust depth-buffer sampling (percentile-based "nearest", median+EMA "center", inverting device depth through the VIEWPORT scale/translate instead of assuming a clean half-range remap, and correcting the lateral bias of reading a PER-EYE depth buffer at an undisplaced screen position), the auto-convergence solve that no longer needs to co-scale separation to lock background disparity, camera-cut detection that snaps rather than eases (why a |z-ema|/ema threshold can never fire on approach, and why the symmetric ratio that fixes it thrashes on close motion unless the snap is approach-only), identifying a reticle you do not draw (position and shape for an anchored crosshair, texture identity when it tracks a target and no position test can name it), asymmetric near/far clamps on depth-shifted overlays (pop-out is not divergence), the rule that convergence must never be used as a scene-depth proxy in thresholds, and the two auto-scale strategies for shifted overlays (cover vs. fit). Engine-agnostic; examples in pseudo-C. Assumes you already have a per-eye off-axis stereo renderer — pairs with a stereo-output skill (SbS/anaglyph/interlaced compositor) for the parts this one doesn't cover.
---

# dynamic3d — auto-convergence and dynamic crosshair for clip-space stereo 3D

This skill is the **adaptive layer** on top of a working off-axis stereo-3D renderer: given a per-eye projection that supports a convergence shear, how do you (1) parameterize the stereo knobs so they stay meaningful as the game's FoV changes, (2) automatically pull convergence in so close objects don't pop out uncomfortably, and (3) place a crosshair/reticle/HUD element at the depth the player is actually aiming at?

It is written against a production implementation — [UU3D](https://github.com/oneup03/UU3D), a UEVR-based injection stereo mod; see [`src/mods/vr/flat3d/Flat3DAutoConvergence.cpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DAutoConvergence.cpp) for the control loop — that shipped the **eye-separation-in-metres** parameterization first, then migrated to **clip space**. The same clip-space math also ships in [rt64-3D](https://github.com/oneup03/rt64-3D) (`Stereo3D` branch, [`src/render/rt64_stereo_renderer.cpp`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/render/rt64_stereo_renderer.cpp)) for N64 recompilation ports, which is where the emulator/recomp-specific notes in §4.1 and §5.1 come from. The two are algebraically identical — same hyperbola, same convergence behaviour, same near-field limit — but they are not equally usable, and roughly a third of this document exists because of machinery the metres form needed and clip space simply deletes. §2 is the migration path if you are on the old form.

**Prerequisite:** you already have per-eye rendering with a convergence shear (see §1) or an equivalent full-frame shift+crop fallback (see §5.3). This skill does not cover building that renderer or composing the two eyes for display.

---

## 1. Foundation — clip-space separation

### 1.1 Matrix-level: the shear IS the knob

The standard off-axis / parallel-camera stereo frustum adds one horizontal shear term to an otherwise-normal perspective projection, in the `[2][0]` slot (row 2, col 0 in a row-major, post-multiplied convention — adjust indices to your matrix layout). Under the clip-space parameterization that term is not *derived from* the knob, it **is** the knob:

```c
// separation = THE stereo setting (see 1.3 for what it means physically)
// dir        = per-eye sign (see 1.4 — do NOT assume it matches your compositing dir)

float shear = dir * separation;

matrix[2][0] += shear;   // added AFTER building the normal symmetric projection
```

Convergence does not appear here at all. That is the defining property of this form: **the shear is invariant under convergence**, and everything convergence does lives in the matching view-space eye translation instead.

That translation is now a *derived* per-frame value, not a setting:

```c
// tan_half_h = tan(half horizontal FoV) — the game's live value, whatever it is
// conv       = convergence distance, in world units
float eye_baseline = 2.0f * separation * tan_half_h * conv;   // total, both eyes
// apply -/+ eye_baseline/2 along the camera-right axis
```

Two consequences worth internalising before you write any of the rest:

- **The physical eye baseline moves with both FoV and convergence.** It has to, for zero parallax to stay at `conv` while `separation` is held fixed. Anything in your codebase that treats eye separation as a stable physical constant will now see it change — audit those call sites.
- **Nothing else in the projection changes.** FoV, aspect, near/far all flow through untouched, so this composes with the game's own (possibly changing) FoV instead of fighting it.

### 1.2 CPU-side: the equivalent per-eye pixel shift

Not everything goes through the projection matrix — crosshairs, HUD, and UI overlays composited in screen space need the SAME disparity a world vertex at that depth would get, computed directly in pixels:

```c
// eye_w = per-eye viewport width in pixels; z = depth of the element, in the
// same units as conv.
float shift_px(float z, float separation, float conv, float eye_w, float dir) {
    float ratio = (z > 0.0f) ? (conv / z) : 0.0f;   // invalid / at-infinity -> 0
    return dir * separation * (ratio - 1.0f) * (eye_w * 0.5f);
}
```

Note what is *absent* versus the metres form: no `P00`, no FoV term, no unit conversion. Zero at `z == conv` (the element sits on the screen plane); at `z -> infinity` it settles at `separation * eye_w / 2` per eye. This one formula is the workhorse for §5's aim-depth crosshair and overlay placement — everything downstream just calls it with a different `z`.

(If your renderer's sky/background elements need the `z -> infinity` limit and how it interacts with FoV, that's a separate, deeper topic — see the `skybox-stereo-depth` skill.)

### 1.3 What `separation` actually means — and how to bound it

Per-eye NDC x is offset by `±separation` at infinity, and NDC x spans the full screen width over `[-1, +1]`. Therefore:

> **Total background disparity = `separation` × screen width.**

`separation = 0.05` puts objects at infinity 5% of the screen width apart. This is the entire reason to prefer this parameterization: the primary knob is a quantity the user can *see*, expressed in the same terms as the constraint that actually bounds it.

That constraint is divergence. Uncrossed disparity larger than the viewer's IPD forces the eyes to turn outward, which cannot be fused at any comfort level:

```
separation_max ~= IPD / screen_width     // ~63mm / ~597mm ~= 0.105 on a 27in 16:9
```

Useful cross-check: this matches NVIDIA's own definition. In 3Dmigoto's `nvstereo.h` the value handed to shaders is `finalSeparation = eyeSeparation * userPercent * 0.01`, i.e. **100% separation is exactly the point where background disparity equals the driver's physical eye separation.** A sane slider therefore runs `0 .. ~0.15` with a default around `0.05`, and on an autostereoscopic panel the value is directly comparable across games as a crosstalk budget.

The canonical shader form of this same math, which any 3Dmigoto/Geo-11-style fix will already be written in:

```hlsl
// StereoParams.Load(0): .x = separation (signed per eye), .y = convergence
output.x += separation * (output.w - convergence);
```

If you expose stereo parameters to injected/override shaders, use that layout verbatim (a `Texture2D<float4>` at `t125` is the established slot) so existing fixes port with no rewrite.

### 1.4 The sign-convention trap: you probably have TWO conventions, not one

This is the single most common source of "my auto-convergence is inverted" or "the crosshair drifts the wrong way" bugs. A real codebase typically has **two independent per-eye sign switches** that must each be verified separately, because they live in different code paths and there is no guarantee they agree:

1. **The projection shear's `dir`** — whatever convention your matrix math / graphics API's clip-space handedness produces. In one verified implementation this came out as `left = -1, right = +1` — the proof required tracing an intermediate reference implementation's own tangent convention (`{l = -tan+o, r = tan+o}`) through a `[2][0] = (l'+r')/(l'-r')` remap; it was **not** obvious from first principles and got the sign wrong on the first attempt. It does agree with 3Dmigoto, whose params texture stores the left eye's separation negated (`leftEye[0] = -finalSeparation`), which is a cheap external cross-check.
2. **The screen-space compositor's `dir`** (used in §1.2, §5) — driven by whichever eye index your compositor iterates as "eye 0" and which half of the output that lands in. In the same implementation this came out as `left(eye 0) = +1, right(eye 1) = -1` — the **opposite** convention from #1, because it answers a different question ("which pixel column offset moves this eye's copy toward/away from center") than the shear does ("which direction bends this eye's frustum").

**Do not assume these two `dir`s share a sign just because they're both "left vs right."** Determine each empirically, independently:

- **Shear sign check:** put an object clearly nearer than convergence on screen. It must show **crossed** disparity — the left-eye image displaced to the *right* relative to the right-eye image (cross your eyes to fuse it; if you instead need to diverge, the sign is inverted).
- **Compositor/overlay sign check:** with the dynamic crosshair enabled, move the aim point from far to near. The overlay must shift in the SAME apparent direction as world geometry at that depth. If they move opposite ways, the compositor `dir` (not the shear `dir`) is inverted.

Treat these as two separate unit tests. A codebase that only checks "does stereo look roughly right" can ship with the compositor sign backwards while the world shear is correct — the symptom is subtle (crosshair drifts wrong, background pop feels off) rather than "3D is broken," which is exactly why it survives review.

There is a **third** place this bites, covered in §6.1: once `dir` is folded into a precomputed constant, the sign of the resulting shift no longer tells you whether a sample is in front of or behind the screen plane.

---

## 2. Migrating from the eye-separation-in-metres parameterization

If you already have a working renderer whose knob is a physical eye separation, this section is the whole conversion.

### 2.1 What the old form looked like

```c
// OLD — separation stored in metres, convergence in metres, plus a third
// "reference FoV" slider needed to keep the first one behaving under zoom.
float fov_scale = tan_half_game / tan_half_ref;
fov_scale_ema  += (fov_scale - fov_scale_ema) * 0.2f;     // hides its own settling
float sep_eff   = base_separation_m * fov_scale_ema;

float o     = (sep_eff * world_scale) * 0.5f / conv;
float shear = dir * o * P00;                               // P00 = 1/tan_half_game
```

Expand it. `sep_eff` multiplies by `tan_half_game / tan_half_ref`, and `P00` divides by `tan_half_game`. **`tan_half_game` cancels exactly.** The entire FoV auto-scale block was multiplying in a term the projection then divided straight back out, at the cost of an EMA that made depth briefly wrong after every hard FoV cut. What survives the cancellation is:

```
shear = dir * base_separation_m * world_scale / (2 * conv * tan_half_ref)
```

which is a constant — and that constant is precisely the clip-space `separation`.

### 2.2 The conversion

```c
separation = base_separation_m / (2.0f * conv_m * tan(radians(reference_fov) * 0.5f))
           * world_scale;
```

This is **exact**, not approximate: it reproduces the user's existing image pixel-for-pixel at their saved convergence. Run it once on config load, gated on the new key being absent while an old one is present, so it fires exactly once and never touches an already-converted profile. Log it.

Fold `world_scale` in if — as is likely — your engine had a world-scale multiplier applied to separation. Under clip space it becomes a *second, invisible* separation multiplier that breaks the "separation = on-screen disparity fraction" invariant, so retire it in this mode (force the accessor to return 1) and absorb the saved value into the conversion. That way nobody's picture changes and the invariant holds.

### 2.3 Per-site checklist

Every one of these is a place the old form leaked FoV or unit terms that now vanish:

| Site | Old | New |
|---|---|---|
| Projection shear | `dir * (sep_m/2/conv) * P00` | `dir * separation` |
| Eye-space shear coeff (for a symmetric-projection compat path) | `sep_m * 0.5 / conv` | `separation * tan_half_h` |
| View translation | the stored `sep_m` | derived `2 * separation * tan_half_h * conv` |
| Overlay pixel shift | `(sep_uu*P00/2) * (1/z - 1/conv) * eye_w/2` | `separation * (conv/z - 1) * eye_w/2` |
| Any "px per (1/z - 1/conv)" constant | `sep_uu * P00 / 4 * eye_w` | `separation * conv_uu / 2 * eye_w` |
| Auto-convergence input | `sep_uu` + `P00` | `separation` |
| FoV auto-scale block | present | **delete** (§3) |
| Reference-FoV slider | present | **delete** (§3) |
| Auto-convergence `depth_scale` output | present | **delete** (§4.3) |

Verify numerically rather than by eye: for a grid of (old sliders × FoVs), the shear, the overlay pixel shift, every derived constant, and the effective eye baseline must all agree between old and new to floating-point epsilon. In the reference migration they matched to ~1e-14 across 30+ combinations, which is what turned "I think this is equivalent" into "this is equivalent."

### 2.4 The one behaviour that genuinely changes

**Convergence stops changing background depth.**

- Old: background disparity was `sep/conv`, so pushing the screen plane out *reduced* the whole image's depth. The two knobs were coupled.
- New: background disparity is `separation`, full stop. Moving convergence only moves what sits in front of the screen.

This is the point of the change, but it removes an accidental safety coupling: a user pushing convergence out now consumes *more* total depth budget than before, not less. Auto-convergence still bounds the near field, so the exposure is manual-mode only — but call it out in release notes, because it is the one thing a returning user can mistake for a regression.

Two knock-on effects to expect:

- **A near-plane clamp on convergence loses its original justification.** In the old form the shear was `sep/2/conv` and genuinely exploded as convergence approached zero. It is now just `separation`. Keep the clamp as a *comfort* limit if you like, but fix the comment — the next reader will otherwise trust a stale numerical rationale.
- **Raising the convergence ceiling becomes safe, and therefore tempting.** It also makes previously-unreachable code paths reachable. See §6 — both bugs documented there were latent for years and only surfaced when a 5m cap became 25m.

---

## 3. FoV changes: there is nothing left to compensate

Under the metres form, screen disparity scaled with `sep * P00 = sep / tan_half`, so a zoom (ADS, sniper scope, cutscene FoV pull) visibly inflated the 3D effect unless you compensated — hence the reference-FoV slider, the `tan_half_game / tan_half_ref` term, and an EMA to hide its settling.

**`separation` is a clip-space quantity. It is FoV-independent by construction.** Delete all of it:

- the reference-FoV slider (it has nothing left to calibrate),
- the `fov_scale` computation,
- the EMA and its file-scope state.

You also get a latent bug fix for free: the EMA meant a hard FoV cut took several frames to settle, during which depth was measurably wrong. The clip-space form is exact on the first frame.

**The residual design question.** FoV-independence is a *choice*, not a law. Physically, a real telephoto lens does magnify disparity, so the metres form was the faithful one and clip space is the comfortable one. If you want depth to respond to zoom, you must now opt *in* — multiply `separation` by something FoV-derived deliberately — rather than getting it by default and compensating it away. For games, comfort is almost always the right default: a scope zoom that also triples your disparity is a fusion problem, not a feature.

**Convergence needs no FoV term either**, and never did — it is a distance, not a disparity.

---

## 4. Auto-convergence — depth-buffer-driven control loop

Goal: keep the **nearest significant on-screen object's** pop-out disparity under a comfort budget, without the player manually chasing convergence, and without the background silently going flat when the pull-in engages.

### 4.1 Robust depth sampling (do this before any control loop)

Two different depth queries feed two different consumers, and both need outlier rejection — a single hot/cold depth texel (particle, muzzle flash, a sliver of geometry at the screen edge) must not yank the control loop or the crosshair:

```c
// "Center" depth (crosshair aim point): small patch (e.g. 8x8-16x16 px) at
// screen center, spatial median, THEN temporal EMA.
float center_z = median(patch_samples);
if (center_ema <= 0) center_ema = center_z;
else if (fabs(center_z - center_ema) / center_ema > 0.01f)   // relative deadband
    center_ema += (center_z - center_ema) * 0.25f;

// "Nearest" depth (auto-convergence input): a wide region-of-interest
// (e.g. the middle ~85% x ~90% of the frame, to exclude HUD-hugging edge
// geometry), robust low-PERCENTILE, not literal min. nth_element around the
// ~2nd percentile index rejects single-pixel spikes while still tracking
// "the nearest object that actually covers pixels."
sort_or_nth_element(z_samples);
float nearest_z = z_samples[percentile_index];   // e.g. index = count/50
```

**Sample the region the viewer SEES, not the whole render target.** The two stop being the same the moment anything crops between render and display — the centred 16:9 slice a compositor takes from a wider-than-16:9 eye on an ultrawide is the common case, and there the discarded margin is about a quarter of the target at each side. An ROI laid out as a percentage of the *target* then reaches well outside the frame, and because the statistic is a low percentile over "nearest", off-screen scenery is exactly the kind of thing that wins it. Convergence pulls in for something the player cannot see and has no way to explain.

Derive the sampling span from whatever owns the crop rule rather than reimplementing it, so the two cannot drift: the loop's job is to agree with the displayed image, and a second copy of the rule is a second thing to keep in step.

Reversed-Z / infinite-far conventions are common in modern engines — convert device depth to view-space Z with `z_view = near_z / device_depth` (verify against a known object at a known distance; don't assume).

**There may be a VIEWPORT transform between NDC and what the buffer holds, and it is probably not a clean half.** Two layers have to be undone, and missing either one produces a plausible-looking result rather than an obviously broken one:

1. *The NDC convention.* A GL-style projection puts ndc.z in `[-1,1]` while the buffer stores `[0,1]`. Feeding device depth straight into the inversion overestimates distance by nearly 2x across a typical range.
2. *The viewport's own depth scale and translate.* Fixed-function pipelines commonly carry a scale that is close to but not exactly 0.5 — an N64 title measured `0.499023` (32704/65536, a G_MAXZ fixed-point value), and consoles that store viewport depth in fixed point generally will.

```c
// Read the scale and translate the frame was actually rendered with.
float ndc_z = (vp_scale_z > 1e-6f) ? ((device_depth - vp_translate_z) / vp_scale_z)
                                   : ((2.0f * device_depth) - 1.0f);
```

That 0.2% discrepancy sounds ignorable and is not, because it perturbs the term that survives after almost all of `m22` cancels. Measured against a real projection (near 4, far 10096) assuming 0.5:

| True distance | Computed | Error |
|---|---|---|
| 10 | 9.97 | −0.3% |
| 100 | 95.6 | −4.5% |
| 1000 | 804 | **−20%** |

**Learn the error signature**: near objects correct, everything beyond reading progressively *closer* than it is. Symptoms are a crosshair that tracks fine up close and sits short of mid-range targets, and auto-convergence that is quietly over-aggressive because it believes the scene is nearer than it is.

**Sampling a PER-EYE depth buffer needs an eye correction.** This one is easy to miss entirely, because the sampling code reads correct and the bug is in the *frame of reference*. In a per-eye renderer there is no centre view: everything in the left eye's depth buffer is displaced horizontally by the disparity for its own depth. An aim point expressed in undisplaced screen space therefore reads a column of the scene offset to one side.

The error grows with distance (disparity does) and it is **one-sided** — an object is detected earlier when approached from one direction than the other, which is how it usually gets reported. Two fixes:

```c
// (a) Compensate for the eye you are sampling. Uses the previous frame's depth,
//     since the shift depends on the depth being measured; it settles in a frame
//     or two and the result is EMA-smoothed anyway.
float aim_x_corrected = aim_x + shift_px(prev_z, separation, conv, eye_w, dir_of_sampled_eye);

// (b) Sample BOTH eyes and take the FARTHER value. Costs a second sample and
//     needs both buffers reachable (a Texture2DArray slice, say), but needs no
//     previous-frame estimate and additionally guards against one eye seeing
//     round an occluder the other does not.
sampled_z = max(z_from_left_eye(x + off), z_from_right_eye(x - off));
```

Reuse the *same* function that shifts the overlay to compute the correction in (a). Deriving a second expression for the same displacement is how the two signs drift apart — see §1.4.

### 4.2 Temporal median prefilter, then asymmetric EMA

Before smoothing, keep a short history (~5 frames) of the raw nearest-depth sample and take its median — a second, independent outlier filter in *time* rather than space, catching single-frame spikes the spatial percentile alone won't.

```c
float z_median = median(last_5_nearest_z_samples);

float rel = fabs(z_median - z_ema) / z_ema;
if (rel > 0.005f) {                                    // relative deadband
    float alpha = (z_median < z_ema) ? 0.20f : 0.05f;  // fast toward camera, slow away
    z_ema += (z_median - z_ema) * alpha;
}
```

Asymmetric rates matter perceptually: an object suddenly getting closer should pull convergence in quickly (comfort), but an object receding should relax slowly (avoid convergence "chasing" every small recession and reading as swimmy). The deadband must be **relative**, not a fixed-unit threshold — a fixed one is simultaneously too twitchy up close and too sluggish at range.

**`fabs(z - ema) / ema` is bounded above by 1 in the approach direction.** Fine for a small deadband, and a silent trap for anything else built on it. When `z < ema` the expression is `1 - z/ema`, which cannot reach 1 no matter how close the object gets; when `z > ema` it is unbounded. So any threshold at or above 1 fires **only for recession** — the direction with nothing to protect against.

A shipped implementation used `rel > 1.5` to detect camera cuts and consequently could never detect a cut *to* a closeup, which is the only case the detector existed for. It looked reasonable in review and the asymmetry is invisible until you write out the algebra. For any large threshold, use a ratio instead — and make it **directional**, not symmetric, for the reason in §4.5:

```c
float approach_ratio = (z < ema) ? (ema / z) : 1.0f;   // unbounded on approach
if (approach_ratio > 2.5f) { /* ... */ }               // 2.5 mirrors the old rel > 1.5
```

### 4.3 The control law

**In the metres form this section contained a bug and its fix.** The naive solve — pick a convergence that puts the nearest object at the target disparity, in isolation — ignored that convergence is global: pulling it toward the player also grew every far object's disparity (`sep/conv`), so the background visibly popped exactly when the near-object protection engaged. The fix was to co-scale separation by `conv_target / manual_conv`, which held background disparity constant by construction, and to emit that ratio as a `depth_scale` output the caller had to apply.

**Under clip space that entire mechanism is unnecessary.** Background disparity is `separation`, which convergence does not touch. There is nothing to lock, so there is no co-scaling and no `depth_scale`. The loop reduces to a single closed form:

```c
// Pop-out (crossed) disparity of the nearest object at a given convergence,
// as a fraction of eye width — POSITIVE when nearer than the screen plane:
//     d(conv) = (separation / 2) * (conv/z - 1)
float half_sep = separation * 0.5f;

float conv_target  = manual_conv;
float d_at_manual  = half_sep * (manual_conv / z_near_ema - 1.0f);

if (d_at_manual > target_disparity) {
    conv_target = z_near_ema * (1.0f + 2.0f * target_disparity / separation);
    conv_target = min(conv_target, manual_conv);    // manual value is a CEILING
}
conv_target = max(conv_target, near_clamp);
```

If you are migrating, the old and new closed forms are the same expression: substituting `k = separation * conv / 2` into `conv = z * (1 + target*manual/k)` yields `conv = z * (1 + 2*target/separation)` exactly. Behaviour is unchanged; only the scaffolding goes away.

Guard on `separation > 0` rather than on `P00 > 0` — the latter no longer appears, and a zero separation would otherwise divide by zero.

### 4.4 Smooth in `1/convergence` space, not convergence space

Disparity is linear in `1/conv`, not in `conv`. A plain per-frame lerp of `conv` accelerates perceptually as convergence gets small — the classic "auto-convergence lunges the last bit of the way" failure. Smooth the reciprocal:

```c
float target_inv = 1.0f / conv_target;
// Asymmetric for the same reason the depth EMA is (§4.2): closing the gap
// protects comfort and should be prompt, easing back out has nothing to protect
// against and reads as swimmy if it hurries. A LARGER target reciprocal is a
// NEARER convergence, so that is the direction to hurry.
float smoothing = (target_inv > inv_conv_smoothed) ? 0.14f : 0.06f;
inv_conv_smoothed += (target_inv - inv_conv_smoothed) * smoothing;
convergence_applied = min(1.0f / inv_conv_smoothed, manual_conv);
```

Both rates want to be asymmetric, and they are not redundant with each other: the EMA governs how fast the loop believes the scene changed, this one governs how fast the picture follows that belief. Symmetric here is survivable but makes every recession read as the image drifting.

### 4.5 Camera cuts: snap, don't ease

An EMA tuned for continuous motion needs many frames to cross a large step, and a cut to a close framing is exactly when the wrong convergence is least tolerable. Detect the step and jump.

Four things make the difference between this helping and it twitching:

- **Test a ratio, not a relative difference** (§4.2), or it cannot fire on approach at all.
- **Snap on APPROACH only, and let recession ease.** Making the ratio symmetric is the obvious next step once the bound is fixed, and it buys a worse failure than the one it cures. Anything held close to the camera and moving — a pickup animation, an NPC leaning in, a creature circling — swings the near statistic past a 2.5x ratio in *both* directions within a second or two, and a symmetric detector answers every crossing with a hard snap and a smoother reset: in, out, in. That reads as the image thrashing, which alarms a viewer far more than the sluggish convergence it replaced. Recession has nothing to protect against anyway: convergence sitting nearer than the scene needs costs only positive parallax, which is bounded by `separation`, so it belongs on the EMA's slow path. Dropping the recede half also breaks the oscillation on its own — after an approach snap the EMA is sitting at the near value, so the return trip cannot clear the ratio a second time. The price is that a genuine cut from a closeup to a vista eases over roughly a second instead of snapping, which is the comfortable direction to be wrong in, and is why it is the half that gives way.
- **Test the RAW sample, not the median.** A 9-frame median needs 5 frames before it starts to cross a step — latency the smooth path wants and the cut path must not pay. The persistence requirement below is what rejects spikes, so the median is not also needed here.
- **Require persistence, but be stingy with it.** A near statistic sensitive enough to be useful will legitimately halve when an object enters one patch, so a single frame past the threshold is not a cut. Two to three consecutive frames is enough — and remember the readback is already several frames behind the GPU, so every confirmation frame is one more spent at the wrong convergence.

Snap to the median of the **last few raw samples**, not to the long median (which still holds mostly pre-cut values, so the jump only goes part of the way) and not to a single raw sample (which puts the whole shot at the mercy of one frame). Reset the `1/convergence` smoother at the same time, or it eases across the step you just jumped.

### 4.6 Manual-as-ceiling and snap-back

- The manual convergence slider is always the **ceiling** — auto only pulls *closer*. This keeps the feature feeling like protection, not "the slider doesn't work."
- On disable, snap back to the manual value immediately (reset EMAs and history) rather than easing out. Easing reads as unexplained drift after the user just turned the feature off.
- Clamp the auto pull-in floor to the larger of a near-plane-derived clamp and any user-configured minimum.

---

## 5. Dynamic / depth-aware crosshair and overlays

### 5.1 Aim-depth crosshair

Reuse §1.2's `shift_px` at the depth sampled under the aim point (§4.1's "center" query), with a static fallback when the sample is invalid:

```c
float z = adaptive_enabled ? center_depth_ema : -1.0f;
if (z <= 0.0f) z = static_fallback_depth;

float crosshair_shift = shift_px(z, separation, conv, eye_w, dir);
```

**Do not inherit a HUD-depth calibration into this path.** If the codebase already shifts flat UI by a slider, it is tempting — and correct for sign safety — to reuse that pipeline. Reuse the *sign convention*; do not reuse its magnitude constants. Those exist to make two UI paths agree with each other (a perspective-projected HUD versus an orthographic one, say) and are typically matched by eye. This path answers a different question: the reticle has to land on the disparity the **world** gets at that depth, which is fixed by the projection.

One port carried a `1.732` perspective-to-ortho HUD constant into its crosshair shift and put the reticle at 1.73x the world's disparity. The signature is distinctive: correct near the convergence distance (where both the right and the inflated value are zero), visibly too deep in the mid range, and indistinguishable from correct far away (where both have saturated at the background plane). It reads as "the crosshair sits behind what I am pointing at" and survives casual testing at close range.

**Identifying a reticle you do not draw.** In an emulator or recompilation you often cannot tag the draw. Which discriminator works depends on whether the reticle is anchored or free:

- *Position and shape* — a small, roughly square quad near screen centre. The right first choice for the common case of a fixed crosshair, which most first-person games have, and it needs nothing from the renderer beyond vertex positions. Expect to gate it on game state: menus, pause screens and transitions draw reticle-shaped quads near centre too, and shifting one of those to aim depth is what visibly corrupts them. A flag the game already exposes ("first-person camera is live") separates them better than any amount of geometry tuning.
- *Texture identity* (a content hash, in a renderer that has one) is exact and survives motion, so it is the answer when the reticle **tracks a target** rather than sitting at centre — there, no position test can name it. Watch for **animated** reticles: each frame of the animation hashes differently, so it is a SET of identities, and matching only some produces a crosshair that tracks depth on some frames and snaps flat on others, read as flicker rather than as a missing feature.
- *Transform/material tags*, where the renderer has them — but check they actually reach the draw type in question. Screen-space rectangles frequently carry no world transform at all, so a tagging scheme built for 3D geometry silently does not apply to them.

A reticle identified by texture gives you its position for free: read it out of the matched draw rather than asking the game where it is.

Once identified, the draw itself gives you the aim point — read the position out of it and feed the *next* frame's depth sample. One frame of lag is invisible against an EMA-smoothed depth, and it removes any dependence on the game reporting where it is. One trap in that loop: do not gate "is this the reticle" on already having an aim point, or nothing is ever identified, no position is ever reported, and the feature deadlocks silently.

**Report it in the space the sampler reads, and prefer the value the renderer already computed.** The renderer worked out where to *put* that quad — in render-target pixels, through whatever widescreen centering, origin and misalignment its overlay path applies. Publish that number. Anything reconstructed instead has to reproduce the same chain, and the reconstruction is where it goes wrong: rebuilding the position from the game's own low-resolution screen space misses the centering, and the sampled column drifts further from the reticle the wider the display gets.

NDC looks like the clean way out of that and is worse. It is normalized against the framebuffer *viewport*, while a depth sampler works in *target* texels — and in renderers that grow targets without ever shrinking them, and pad them for alignment, those two spans differ by an amount that changes **during play**. The result is not a fixed offset you would spot immediately; it is an aim depth that wanders as the target grows, which reads as the feature having become unstable rather than mis-calibrated. If some correction of yours genuinely is in NDC — a per-eye disparity shift, say — carry the viewport span alongside the point and scale by that, not by the target width.

Two strategies for what actually gets drawn, in increasing order of integration effort:

- **Own reticle ("laser sight"):** draw a small dedicated sprite per eye at the computed shift. Trivial, always available, fully depth-correct, but replaces whatever crosshair the game draws.
- **Game's own crosshair, extracted:** if the game's UI is captured into a texture you composite per-eye anyway, cut a small region out of the flat UI composite around the expected crosshair position — via an alpha/region mask, not a second draw that would double it — and re-composite *that same source region* again at the aim-depth shift. The result is the game's own reticle at the correct depth with the rest of the HUD flat. Caveats: any other center-anchored element inside the mask radius (hit marker, interaction prompt) rides along; games whose crosshair isn't screen-centered, or is drawn into the 3D scene, need the fallback. Make the radius and the mode user-configurable.

### 5.2 Overlay/UI depth placement and the two auto-scale strategies

A full-screen UI layer composited at a shifted depth either clips one edge or leaves a gap on the other unless you also scale it. There are two correct answers depending on WHAT is shifted — conflating them was an actual shipped bug:

**Cover (zoom in to hide the gap)** — for a shift applied to a complete, opaque, full-frame image where there are no source pixels beyond the frame edge to reveal (e.g. §5.3's whole-eye shift):

```c
float scene_scale = (eye_w + 2.0f * fabs(shift_px)) / eye_w;   // >= 1, zooms IN
```

**Fit (shrink to stay fully on-screen)** — for an overlay/UI/HUD layer you composite on top, where the source is its own texture and you'd rather shrink slightly than lose content off an edge:

```c
float overlay_scale = max((eye_w - 2.0f * fabs(shift_px)) / eye_w, min_scale_floor);  // <= 1
```

**The bug this distinguishes:** an earlier version used *cover* for the UI overlay ("the UI should fill the screen, so zoom to cover the shift"). It looked right at small shifts and silently clipped UI content off the outer edge at larger depth values, because a cover zoom on a layer that ISN'T the opaque background pushes real UI content past the viewport, not blank space. Use `fit` for overlays; reserve `cover` for the one case where you are re-covering a hole with nothing behind it. Ask: "is there real content behind this that the shift reveals?" — if yes, `fit`.

Apply the scale about the **horizontal** center only; the disparity is horizontal, so a vertical shrink only letterboxes for no benefit.

**Useful clip-space simplification:** if you express a UI layer's depth as a *multiple of convergence* (`z_ui = gui_depth_factor * conv`), then

```
shift_px = separation * (conv/(gui_depth_factor*conv) - 1) * eye_w/2
         = separation * (1/gui_depth_factor - 1) * eye_w/2
```

— convergence cancels, so the UI layer's shift is **convergence-invariant**, and at `gui_depth_factor == 1` it is exactly zero. This is worth knowing because it lets you rule the UI mapping out as a suspect when debugging convergence-dependent symptoms (see §6.2).

### 5.3 Fallback when true per-pixel shear isn't available

Some output paths can't do a real off-axis shear (a downstream compositor that only accepts symmetric per-eye images, or a display mode requiring zero-shear input). Render at zero shear and let a whole-eye pixel shift do the convergence job:

```c
float scene_shift = shift_px(/* z = infinity */ -1.0f, separation, conv, eye_w, dir);
float scene_scale = (eye_w + 2*fabs(scene_shift)) / eye_w;   // COVER — see 5.2
```

Any overlay composited **before** this whole-frame shift+crop must compute its own shift relative to the frame's bias — the crop map already remaps its coordinates once:

```c
float overlay_final_shift = (shift_px(z_overlay, ...) - scene_shift) / scene_scale;
```

Skipping the `- scene_shift` term double-shifts overlays relative to world geometry.

---

## 6. Clamps and thresholds — two traps that only surface at high convergence

Both of these were latent for years in the reference implementation and became reproducible the week the convergence ceiling went from 5m to 25m. If you widen a range, assume you have just made unreachable code reachable.

### 6.1 Pop-out is not divergence — clamp the two directions differently

Depth-shifted overlays need a safety clamp so a bad depth sample can't fling an icon off-screen. The obvious implementation uses one symmetric limit. That is wrong, and it fails asymmetrically:

- **Behind the screen plane**, the constraint is physical. Uncrossed disparity beyond IPD forces the eyes to diverge and cannot be fused. A per-eye cap of ~0.05 eye-widths (≈ `IPD/screen_width` total) is exactly right.
- **In front**, the eyes converge inward. There is no divergence to protect against, and reusing the same cap merely clips valid pop-out.

With a shared cap, an element saturates once it is nearer than `conv / (1 + 2*cap/separation)` — at `cap = 0.05, separation = 0.05` that is `conv/3`. At 1m convergence nothing realistic hits it; at 5m everything closer than ~1.7m stops tracking depth and sits at a fixed shift. The symptom reads as *"the HUD stopped locking to the right depth"*, not as clipping.

```c
// raw = invz - inv_conv, POSITIVE when the sample is NEARER than the screen plane.
float lim = (raw > 0.0f) ? limit_near : limit_behind;   // e.g. 0.15 vs 0.05
float shift = clamp(s_px / eye_w, -lim, lim);
```

**The sign trap (this is §1.4's third case).** You cannot branch on the sign of the shift itself. Implementations typically fold `dir` into the precomputed per-eye constant, so the shift's sign encodes *which eye*, not near-vs-far — it flips between eyes while the near/far sense does not. Branch on the raw `invz - inv_conv` term, which is eye-independent.

### 6.2 Never use convergence as a scene-depth proxy

**Convergence is a viewer-comfort setting. It carries no information about how far away the scene is.** Any threshold you normalize by it will silently retune itself whenever the user adjusts comfort.

The reference bug: a HUD classifier decided "is this element world-anchored?" partly from lateral camera translation — world-anchored UI should slide via parallax when the camera strafes. To express that motion as a screen fraction it needed a depth, and convergence was the convenient stand-in:

```c
// WRONG — the gate now scales with a comfort setting.
float trans_mag = cam_lateral_delta / conv_uu / (2 * tan_half_h);
bool  translating = trans_mag > floor;
```

Required lateral speed to clear the gate scaled linearly with convergence: ~0.18 m/s at 1m convergence, ~1.8 m/s at 10m. Since only *lateral* motion counts (walking straight forward contributes nothing), ordinary movement stopped clearing it somewhere in the 5-10m band. That removed one of two world-evidence paths, so the classifier fell through to its static branch whenever there was rotation flow, and world-anchored icons decayed back to flat HUD depth. Reported as *"icons hold world depth below 5 convergence and pop back to HUD depth between 5 and 10."*

```c
// RIGHT — a fixed reference depth. Convergence drops out entirely.
float ref_z = world_units_per_metre;              // 1m, or a measured scene depth
float trans_mag = cam_lateral_delta / ref_z / (2 * tan_half_h);
```

Pick the reference so it reproduces the previous behaviour at your *default* convergence (here, 1m matched a 1m default), and existing tuning carries over untouched.

**Generalise the rule:** grep for every division by convergence outside §1.1's projection math and §4's control loop. Each one is either a genuine disparity computation or this bug. A measured scene depth is a valid proxy; a comfort slider never is.

---

## 7. Pitfalls

1. **Assuming one sign convention covers the whole codebase.** §1.4 — verify the projection shear sign and the compositor shift sign independently; they are not guaranteed to match. And see §6.1 for the third case, where a folded-in `dir` makes the shift's sign useless as a near/far test.
2. **Keeping the FoV auto-scale machinery after migrating.** §3 — under clip space it multiplies in a term the projection immediately divides out, and its EMA makes depth wrong for several frames after every FoV cut. Delete it and the reference-FoV slider.
3. **Keeping the auto-convergence `depth_scale` co-scaling.** §4.3 — it existed only to lock background disparity, which clip space locks by construction.
4. **A shared clamp on depth-shifted overlays.** §6.1 — pop-out is not divergence; a symmetric cap silently stops near content tracking depth at high convergence.
5. **Convergence used as a stand-in for scene depth.** §6.2 — any threshold normalized by convergence retunes itself when the user changes a comfort setting.
6. **Linear smoothing of convergence.** §4.4 — smooth `1/convergence`, or the loop lunges as convergence gets small.
7. **Absolute-unit deadbands on depth EMAs.** §4.2 — use a relative deadband.
8. **Literal `min()` for "nearest depth."** §4.1 — one stray texel reads as "an object is 2cm away" and slams convergence to the floor. Low percentile over a wide ROI.
9. **Cover-scaling a translucent overlay.** §5.2 — cover only when there is truly nothing behind the shifted layer.
10. **Treating the eye baseline as constant.** §1.1 — under clip space it is derived and moves with both FoV and convergence. Audit anything that assumed otherwise (head tracking, IPD readouts, anything deriving a physical camera offset).
11. **Leaving a world-scale multiplier live.** §2.2 — it becomes a second, invisible separation multiplier that breaks the "separation = screen-width fraction" invariant.
12. **Widening a range without auditing what that makes reachable.** §6 — both bugs there were latent until a 5m cap became 25m.
13. **Overlay math computed relative to the raw frame instead of the post-crop frame** when using the whole-frame-shift fallback. §5.3 — forgetting `- scene_shift` double-shifts overlays.
14. **Assuming the depth buffer holds ndc.z.** §4.1 — there is often a viewport scale and translate on top, and it is rarely exactly 0.5. Near reads correct while distance reads progressively short.
15. **Sampling a per-eye depth buffer at an undisplaced screen position.** §4.1 — the sample is laterally biased, worst at distance, and asymmetric: objects are detected earlier from one side than the other.
16. **Building a large threshold on `fabs(z - ema) / ema`.** §4.2 — bounded by 1 on approach, so it fires only for recession. Use a ratio.
17. **Inheriting HUD calibration constants into the aim-depth shift.** §5.1 — the crosshair must match world disparity, not a UI convention. Reuse the sign, never the magnitude.
18. **Identifying a reticle that MOVES by its position.** §5.1 — fine for a screen-anchored crosshair, useless for one that tracks a target. Texture identity survives the motion; an animated reticle is a set of hashes, not one.
19. **An ROI measured against the render target when something crops before display.** §4.1 — the sampler reads scenery the viewer never sees, and "nearest" is the statistic most likely to pick it up.
20. **Publishing the reticle position in NDC to a consumer working in target texels.** §5.1 — viewport and target are different spans, and in renderers that grow targets the difference moves during play, so it presents as instability rather than as an offset. Publish the renderer's own pixel placement.
21. **A symmetric cut detector.** §4.5 — curing the approach bound by making the ratio symmetric costs more than it buys: anything oscillating close to the camera then snaps in and out repeatedly. Snap on approach, ease on recession.

---

## 8. Quick reference

```c
// --- §1 core -----------------------------------------------------------------
shear         = dir * separation;                       // matrix[2][0]; NO conv term
eye_baseline  = 2 * separation * tan_half_h * conv;     // derived, per frame
shift_px(z)   = dir * separation * (conv/z - 1) * eye_w/2;
// separation == total background disparity as a fraction of screen width.
// ceiling ~= IPD/screen_width (~0.105 on 27in 16:9); sane slider 0..0.15, default 0.05.

// --- §2 migration from separation-in-metres ----------------------------------
separation = sep_m / (2 * conv_m * tan(reference_fov/2)) * world_scale;   // exact
// then DELETE: fov_scale + its EMA, the reference-FoV slider, autoconv depth_scale,
// and the world-scale multiplier (fold its saved value into the conversion above).

// --- §4 depth inversion --------------------------------------------------------
ndc_z  = (device_depth - vp_translate_z) / vp_scale_z;  // NOT 2*d-1; vp scale is
view_z = m32 / (-ndc_z - m22);                          //   rarely exactly 0.5
aim_x += shift_px(prev_z, ..., dir_of_sampled_eye);     // per-eye buffer correction

// --- §4 auto-convergence ------------------------------------------------------
z_near     = robust_percentile(depth_roi);              // NOT min()
z_near_ema = asymmetric_ema(median5(z_near), rel_deadband=0.005, near=0.20, far=0.05);
approach   = (z < ema) ? ema/z : 1;                     // RATIO: |z-ema|/ema is
if (approach > 2.5 for 2+ raw frames) snap(median3);    //   capped at 1 approaching
// APPROACH ONLY. A symmetric test thrashes on anything oscillating up close.
d_manual   = (separation/2) * (manual_conv/z_near_ema - 1);
conv_target = (d_manual > target_disparity)
            ? clamp(z_near_ema * (1 + 2*target_disparity/separation), near_clamp, manual_conv)
            : manual_conv;
inv_conv_ema = ema(inv_conv_ema, 1/conv_target, near=0.14, far=0.06);  // reciprocal,
conv_applied = min(1/inv_conv_ema, manual_conv);        //   asymmetric like the z ema

// --- §5 crosshair / overlays ---------------------------------------------------
crosshair_shift     = shift_px(center_depth_ema_or_static_fallback);
// Sign convention may be shared with the HUD path; its MAGNITUDE constants never are.
overlay_scale_fit   = max((eye_w - 2*abs(shift))/eye_w, floor);   // translucent layers
overlay_scale_cover = (eye_w + 2*abs(shift))/eye_w;               // opaque whole-frame only
// UI at z = factor*conv  =>  shift = separation*(1/factor - 1)*eye_w/2, conv cancels.

// --- §6 clamps ------------------------------------------------------------------
lim   = (invz - inv_conv > 0) ? 0.15 : 0.05;            // near vs behind, NOT symmetric
shift = clamp(s_px/eye_w, -lim, lim);                   // branch on raw, not on sign(s_px)
```

### Decision table

| I need to... | Use |
|---|---|
| Pick the primary stereo knob | §1.3 `separation` = background disparity / screen width |
| Convert an existing metres-based renderer | §2.2 conversion + §2.3 checklist |
| Keep 3D depth constant while the game zooms | Nothing — §3, it's free |
| Stop close objects popping out uncomfortably | §4 auto-convergence (no `depth_scale`) |
| Put a crosshair on the thing being aimed at | §5.1, `shift_px` at sampled center depth |
| Composite a full HUD/menu at a chosen depth | §5.2 `fit` scale |
| Fake per-eye shear on a symmetric-only output path | §5.3 whole-frame shift + `cover` scale |
| Clamp a depth-shifted overlay safely | §6.1 asymmetric near/behind limits |
| Debug "it breaks at high convergence" | §6.2 — grep for divisions by convergence |
| Verify a sign is right before shipping | §1.4, two independent tests (three with §6.1) |
| Work out why depth "stops working" past some distance | Usually not a bug — disparity saturates hyperbolically, so past ~2-3x convergence everything is near the background plane whatever its real depth. Check whether the distance where it stops is around that before hunting a sampling failure. A smooth compression is the curve; an abrupt snap is a failing sample. |
| React faster to a camera cut | §4.5 — ratio test on the raw sample, short persistence, snap the EMA and reset the reciprocal smoother |

---

## 9. What this skill does NOT cover

- **Building the per-eye off-axis projection matrix itself**, or composing two eye images for display (SbS/TaB/interlaced/anaglyph/autostereo weaver) — see `output3d` (engine-agnostic) or the engine-specific sibling `rt64-stereo-3d`.
- **Sky/background elements at (near-)infinite depth** and the FoV-compensation cancellation trap specific to that limit — see `skybox-stereo-depth`.
- **Head tracking.** This skill assumes a fixed viewer; head-coupled perspective is a separate offset applied to the eye positions before any of this math runs. Note §1.1's warning: the derived eye baseline is no longer constant, which matters if you mix the two.
- **How to obtain a depth buffer in the first place** (engine hooks, render-target interception, compute-shader histograms). This skill assumes you can already get *a* per-pixel depth sample, and covers only what to do with it.
- **Autostereoscopic panel weaving (LeiaSR etc.)** — see `leiasr-integration`. Note that `separation` from §1.3 doubles as a directly comparable crosstalk budget on such panels.
- **Injecting stereo into shaders you don't control** (3Dmigoto/Geo-11-style fixes). §1.3 gives the parameter layout to expose; the fix-authoring workflow itself is out of scope.
