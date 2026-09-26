---
name: extraoutput3d
description: The stereo-3D output paths and variants that `output3d` defers — either because they need real OS-level plumbing or because they're niche variants you shouldn't reach for by default. (1) HDMI 1.4 frame-packed (720p60 / 1080p24 / 1080p60) — TaB-with-a-spec-mandated-blank-gap in the compose shader PLUS actually programming the display into the frame-packing video timing (a custom mode not normally exposed) via NvAPI / AMD ADL / CRU-ChangeDisplaySettingsExW, with the exact HDMI-3D timing table (1920×2205, 1280×1470, pixel clocks) and auto-apply/revert lifecycle. (2) Dual-monitor Dual-Display (+ vertically-flipped variant for mirror rigs) — spans the swap chain across the whole virtual desktop (SM_C*VIRTUALSCREEN), routes left monitor = L eye / right = R, and rides `output3d`'s automatic full-SbS native-render+upscale path because a two-16:9-monitor span is ~32:9. (3) The seven anaglyph variants beyond Red-Cyan Dubois — Green-Magenta Dubois (same technique, rarer coloring), simple channel-swap, iaian7 deghosted/contrast (with the two live-tunable Contrast/DeGhost knobs), half-color compromise, and Blue-Amber — which output3d keeps out of its default set (it ships only Red-Cyan Dubois, since red/cyan is the coloring people actually own); use these only when a user explicitly asks. Assumes you already have `output3d`'s shared compose/repack shader, SampleEye(), Anaglyph() dispatch + HDR normalize wrapper, per-eye texture cache, and full-SbS auto-detection — this skill is only the deferred paths that pair with those, not the stereo math (`dynamic3d`) or the core modes (`output3d`).
---

# extraoutput3d — deferred output paths & niche variants

This is a companion to the **`output3d`** skill. `output3d` owns the two-pass per-eye render loop, the persistent per-eye texture cache, the one shared fullscreen compose/repack shader, the automatic full-SbS detection, DPI awareness, the upscale-then-pattern rule, and the single recommended anaglyph mode (Red-Cyan Dubois). This skill covers the paths and variants it deliberately defers — for one of two reasons:

- **They need OS-level plumbing** beyond a compose-shader branch (custom display timings; multi-monitor window spanning) that the core skill leaves out to stay engine/API-agnostic.
- **They're niche variants** that shouldn't be the default — output3d ships only Red-Cyan Dubois anaglyph; the other seven anaglyph variants (Green-Magenta Dubois plus six non-Dubois techniques) live here so a reader isn't nudged toward them.

The three families:

- **Frame-packed** (§2) = TaB + a spec-mandated blank gap in the shader, **plus** actually driving the display into the HDMI-1.4 frame-packing video timing (a mode Windows won't expose on its own).
- **Dual-display** (§1) = the SbS repack, **plus** spanning one swap chain across two physical monitors and positioning it over the virtual desktop.
- **The seven anaglyph variants beyond Red-Cyan Dubois** (§3) = pure compose-shader branches (no OS plumbing) — Green-Magenta Dubois (rare coloring) plus the six non-Dubois techniques — separated out only because Red-Cyan Dubois is the right default and these are opt-in. **Recommend Red-Cyan Dubois (in `output3d` §4) unless a user explicitly asks for one of these.**

The first two are, at the pixel level, just SbS/TaB repacks — so they reuse `output3d`'s `SampleEye()` and per-eye cache unchanged; what makes them "extra" is everything *around* the shader. The third reuses `output3d`'s `Anaglyph()` dispatch and its HDR normalize/re-encode wrapper unchanged — only the per-variant matrix/accumulate math differs.

It assumes you already have, from `output3d`:
- The shared compose shader with the `mode`-keyed `Repack()` switch and `SampleEye(half_idx, u, v)` (which already handles `eye_swap` and the symmetric-projection convergence fallback).
- The persistent per-eye texture cache (AFR-aware).
- **Automatic full-SbS detection** (`output3d` §8): output aspect ≥ ~2.5 with an SbS-family mode ⇒ render each eye at native per-eye width and upscale to full output width, no display un-squish. Dual-display leans on this directly.
- **Per-monitor-DPI awareness** set process-wide before the swap chain exists (`output3d` §3.1). Frame-packed needs this too — a DPI-virtualized window can't present a pixel-exact 2205-row frame.

---

## Mode enum placement

In the shipped implementation ([UU3D](https://github.com/oneup03/UU3D), [`src/mods/vr/flat3d/Flat3DShaders.hpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DShaders.hpp)) all of these modes sit in the same `Flat3DOutputMode` enum as the core modes — they are not a separate subsystem at the enum level, only at the plumbing/default-set level:

```cpp
enum class Flat3DOutputMode : int32_t {
    SBS = 0, TAB, ROW_INTERLACED, COL_INTERLACED, CHECKERBOARD, LEIA_SR,
    FRAMEPACKED_720P60,   // HDMI 1.4 TaB + 30-row blank gap  → 1280×1470 timing
    FRAMEPACKED_1080P24,  // HDMI 1.4 TaB + 45-row blank gap  → 1920×2205 @24
    FRAMEPACKED_1080P60,  // HDMI 1.4 TaB + 45-row blank gap  → 1920×2205 @60
    DUAL_DISPLAY,         // horizontal split spanning two monitors (left = L)
    DUAL_DISPLAY_FLIP,    // as DUAL_DISPLAY but the LEFT image is vertically flipped
    KATANGA,
    ANAGLYPH_RC,          // §3 — simple channel-swap, red/cyan
    ANAGLYPH_RC_DUBOIS,   // (stays in output3d §4 — the recommended default)
    ANAGLYPH_RC_DEGHOSTED,// §3 — iaian7 contrast/deghost, red/cyan
    ANAGLYPH_RC_COMPROMISE,// §3 — half-color, red/cyan
    ANAGLYPH_GM,          // §3 — simple channel-swap, green/magenta
    ANAGLYPH_GM_DUBOIS,   // §3 — Dubois, green/magenta (rare coloring, deferred here)
    ANAGLYPH_GM_DEGHOSTED,// §3 — iaian7 contrast/deghost, green/magenta
    ANAGLYPH_BLUE_AMBER,  // §3 — iaian7 technique, blue/amber
    COUNT,
};
```

Only **Red-Cyan Dubois** stays documented in `output3d` §4, because it's the recommended default; the other seven anaglyph modes — Green-Magenta Dubois (rare coloring) and the six non-Dubois variants — are documented here (§3). They're all the same enum and the same `Anaglyph()` dispatch — the split is editorial (which to reach for), not architectural.

Keep the integer contract in sync across the three places that duplicate it (native enum, HLSL `k*` constants, UI display-name list) exactly as `output3d` warns — adding a mode here means updating all three.

---

## 1. Dual-display (two physical monitors, one stereo pair)

The output window spans **two side-by-side monitors**; the left monitor shows the left eye, the right monitor the right eye. You wear no glasses — the two monitors *are* the two eyes (a mirror-rig / Wheatstone-viewer / beamsplitter setup, or just two panels you cross- or parallel-view). `DUAL_DISPLAY_FLIP` vertically flips the left image for rigs where the left monitor is mounted upside-down behind a mirror.

### 1.1 Span the swap chain across the virtual desktop

A single-monitor swap chain can't cover two displays. Size the swap chain **and** its window to the whole virtual-desktop bounding box, and position it at the virtual-desktop origin:

```cpp
if (mode == DUAL_DISPLAY || mode == DUAL_DISPLAY_FLIP) {
    swap_w = win_w = GetSystemMetrics(SM_CXVIRTUALSCREEN);  // full multi-mon width
    swap_h = win_h = GetSystemMetrics(SM_CYVIRTUALSCREEN);
    target_x = GetSystemMetrics(SM_XVIRTUALSCREEN);          // origin can be negative
    target_y = GetSystemMetrics(SM_YVIRTUALSCREEN);
}
```

Key consequence: **a spanning swap chain has no single physical display mode to force.** Unlike every other mode, you cannot call "apply this resolution/refresh to the output" — there isn't one output. Present windowed/borderless over the span; never try to go exclusive-fullscreen (it would snap to one monitor). This also means the two monitors' own modes (resolution, refresh) are whatever the OS already has them at — arrange and match them in display settings first.

### 1.2 Compose — it's the SbS repack

Plain `DUAL_DISPLAY` is literally `output3d`'s `kSbS` branch: left half of the spanning output = left eye, right half = right eye. Route it through the same branch — no new code:

```hlsl
if (m == kSbS || m == kLeiaSR || m == kDualDisplay) {
    int   half_idx = (uv.x < 0.5) ? 0 : 1;
    float u_half   = (uv.x < 0.5) ? (uv.x * 2.0) : ((uv.x - 0.5) * 2.0);
    return SampleEye(half_idx, u_half, uv.y);
}
```

The flip variant is the one extra branch — vertically flip only the left half:

```hlsl
if (m == kDualDisplayFlip) {
    int   half_idx = (uv.x < 0.5) ? 0 : 1;
    float u_half   = (uv.x < 0.5) ? (uv.x * 2.0) : ((uv.x - 0.5) * 2.0);
    float v_src    = (half_idx == 0) ? (1.0 - uv.y) : uv.y;   // left eye flipped
    return SampleEye(half_idx, u_half, v_src);
}
```

### 1.3 Dual-display rides the automatic full-SbS path

Two 16:9 monitors side by side is a ~32:9 span, so a dual-display output almost always clears `output3d`'s full-SbS aspect threshold (≥ ~2.5). That means it automatically takes the **native-render + upscale** path: the engine is fed a half-width client rect so it renders each eye at native per-eye (16:9) width, and the compositor upscales each half to fill its monitor — **no horizontal squish**. Treat the dual-display modes as members of the `sbs_family` set (`SBS`, `DUAL_DISPLAY`, `DUAL_DISPLAY_FLIP`) that drives that detection. See `output3d` §8; you get the un-squish for free by classifying these modes correctly, and there is no separate toggle.

### 1.4 Dual-display pitfalls

| Symptom | Cause |
|---|---|
| One eye lands on the wrong monitor / halves swapped | The two displays aren't arranged strictly left-then-right in OS display settings; the virtual-desktop split assumes monitor A occupies the left half of the span. |
| Image squished on each monitor | Modes not classified into `sbs_family`, so full-SbS auto-detect never fires — see §1.3 / `output3d` §8. |
| Tearing differs between the two monitors | Two independent scanouts under one present; there's no cross-monitor genlock. Match refresh rates and accept that perfect sync needs hardware genlock (Quadro/FirePro Sync). |
| Snaps to one monitor / goes black | Went exclusive-fullscreen on a span — must stay windowed/borderless (§1.1). |

---

## 2. HDMI 1.4 frame-packed

"Frame packing" is the mandatory HDMI 1.4 3D format: the two eyes are stacked (Top-and-Bottom) into **one oversized frame** with a spec-mandated blank *active-space* gap between them, and the sink (TV/projector) de-packs it back into left/right internally. A "1080p" frame-packed signal is not 1920×1080 — it is a **1920×2205** frame (1080 left + 45 blank + 1080 right). 720p60 frame packing is **1280×1470** (720 + 30 + 720).

Two independent pieces, both required:

1. **Compose** the TaB-with-gap image in the shader (easy — below).
2. **Program the display** into the frame-packing video timing (the hard part — this custom mode is not in Windows' normal mode list, so you push it via the GPU vendor API or a CRU-style override).

### 2.1 Compose: TaB + blank active-space gap

Extend `output3d`'s TaB branch. The gap is a fixed number of **output rows** (spec-mandated per format), expressed as a UV fraction of the total frame height; rows inside the gap are black:

```hlsl
if (m == kTaB || m == kFramePacked720p60 || m == kFramePacked1080p24 || m == kFramePacked1080p60) {
    float gap_px = 0.0;
    if      (m == kFramePacked720p60) gap_px = 30.0;   // 720p60
    else if (m != kTaB)               gap_px = 45.0;   // 1080p24 / 1080p60
    float gap    = (gap_px > 0.0) ? (gap_px / (float)out_size.y) : 0.0;
    float half_h = (1.0 - gap) * 0.5;                  // each eye occupies (1-gap)/2

    if (uv.y < half_h)              return SampleEye(0, uv.x, uv.y / half_h);
    if (uv.y >= half_h + gap)       return SampleEye(1, uv.x, (uv.y - half_h - gap) / half_h);
    return float4(0.0, 0.0, 0.0, 1.0);                 // active-space blanking gap → black
}
```

`out_size.y` must be the **real, display-native frame height** (2205 or 1470) for the gap-row count to land on the right rows — which is exactly why the display-timing step (§2.2) and process DPI-awareness (`output3d` §3.1.1) matter: if the backbuffer isn't the true packed height in physical pixels, the gap and the eye split drift.

The gap-row counts (30 for 720p60, 45 for 1080p) are fixed by the HDMI 1.4 3D spec, not free parameters.

### 2.2 Program the display into the frame-packing timing

The compose shader produces the right *pixels*, but the monitor won't interpret them as 3D unless it's actually receiving the frame-packing *video timing*. That timing is a non-standard mode you must install and select yourself. Three mechanisms, in preference order, with a common spec table (a complete implementation of all three lives in UU3D's [`Flat3DDisplayTiming.cpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DDisplayTiming.cpp)):

```cpp
struct FramePackTimingSpec {
    int  active_w, active_h;   // full packed frame, e.g. 1920 × 2205
    int  per_eye_h;            // 1080 or 720
    int  gap_rows;             // 45 or 30
    float refresh_hz;          // 24 or 60
    int  h_total, h_front, h_sync, h_back;   // horizontal blanking
    int  v_total, v_front, v_sync, v_back;   // vertical blanking
    // pixel clock = h_total * v_total * refresh_hz
};

// HDMI 1.4 3D frame-packing mandatory formats:
static const FramePackTimingSpec kSpecs[] = {
    // 720p60 : 1280×1470 @60, 30-row gap. Htot 1650, Vtot 1500 → 148.5 MHz.
    { 1280, 1470,  720, 30, 60.0f,  1650, 110, 40, 220,  1500, 5, 5, 20 },
    // 1080p24: 1920×2205 @24, 45-row gap. Htot 2750, Vtot 2250 → 148.5 MHz.
    { 1920, 2205, 1080, 45, 24.0f,  2750, 638, 44, 148,  2250, 4, 5, 36 },
    // 1080p60: 1920×2205 @60, 45-row gap. Htot 2750, Vtot 2250 → 371.25 MHz (needs HDMI 2.0+ bandwidth).
    { 1920, 2205, 1080, 45, 60.0f,  2750, 638, 44, 148,  2250, 4, 5, 36 },
};
```

Note the pixel clocks: 720p60 and 1080p24 are 148.5 MHz (fit in HDMI 1.4's bandwidth — that's why 1080p frame packing is spec'd at 24 Hz, not 60); 1080p60 frame packing is 371.25 MHz and needs an HDMI 2.0+ link even though "1080p60 3D" is what the user asked for.

**Mechanism A — NVIDIA NvAPI (preferred on NV):** install a custom display/timing via `NvAPI_DISP_TryCustomDisplay` / `NvAPI_DISP_SetCustomDisplay` (or the `NV_TIMING` path from the spec's H/V blanking fields), then apply it. NvAPI lets you register a timing the EDID doesn't advertise, which is what frame packing needs.

**Mechanism B — AMD ADL:** the ADL custom-mode / display-mode-timing-override entry points play the same role on AMD GPUs.

**Mechanism C — CRU / `ChangeDisplaySettingsExW` fallback (vendor-neutral):** if the frame-packing timing has already been added to the driver's mode list out-of-band (e.g. the user ran CRU — Custom Resolution Utility — to add 1920×2205@24), you can select it with a plain `ChangeDisplaySettingsExW(DEVMODE{ dmPelsWidth=1920, dmPelsHeight=2205, dmDisplayFrequency=24, … }, CDS_FULLSCREEN)`. This can't *create* an arbitrary timing (no blanking control), so it only works when the mode already exists — hence the fallback slot.

### 2.3 Lifecycle: auto-apply and always revert

The frame-packing timing is a temporary, unusual desktop mode. Wrap it in a strict apply/restore lifecycle:

- **On entering** a frame-packed mode: save the current display mode, apply the matching `FramePackTimingSpec`, then resize the swap chain to the packed `active_w × active_h`.
- **On leaving** it (mode change, alt-tab loss if you choose, and unconditionally on exit/shutdown): **restore the saved mode.** If you skip this you leave the user's desktop at 1920×2205 — a mode nothing else can display correctly, which reads as a hang or a black desktop. Treat restore as a must-run cleanup path (RAII / guaranteed on the mod's teardown), not a best-effort.

### 2.4 Frame-packed pitfalls

| Symptom | Cause |
|---|---|
| TV shows a stacked 2D image, not 3D | The display timing wasn't installed/selected — the shader is producing correct pixels but the sink never got the frame-packing timing that flags it as 3D (§2.2). |
| Blank gap lands on wrong rows / eyes bleed into the gap | `out_size.y` isn't the true packed height in physical pixels — check swap-chain resize to `active_h` and process DPI-awareness (`output3d` §3.1.1). |
| 1080p60 3D won't apply, 1080p24/720p60 work | 1080p60 frame packing is 371.25 MHz — needs an HDMI 2.0+ link; HDMI 1.4 tops out around 340 MHz. Fall back to 1080p24 or 720p60. |
| Desktop stuck in an unusable mode after exit/crash | Restore path didn't run — see §2.3; make it a guaranteed-cleanup path. |
| CRU fallback "does nothing" | `ChangeDisplaySettingsExW` can only *select* an existing mode; the frame-packing timing must already be in the mode list (added via CRU or the vendor API) — it can't synthesize a custom timing. |

---

## 3. Anaglyph variants beyond Red-Cyan Dubois (opt-in only)

**Recommendation first: use Red-Cyan Dubois (`output3d` §4). Only reach for these when a user explicitly asks** — for the green/magenta glasses coloring (rare), for a specific pair of glasses whose filter leakage the fixed Dubois matrix doesn't match, or for the Blue-Amber coloring. They ship as extra `mode` values in the same enum but are not part of the recommended default set. (output3d keeps **only Red-Cyan Dubois** as the default, because red/cyan is overwhelmingly the coloring people own.)

All seven plug into `output3d`'s `Anaglyph(int mode, float4 cA, float4 cB)` dispatch and its HDR `ToNorm`/`FromNorm` wrapper unchanged — `cA`/`cB` arrive already normalized to ~[0,1] display-referred space (identity in SDR), and every `return` re-encodes through `FromNorm`. Only the per-variant math below differs; add these as extra branches to the same function.

### 3.1 Green-Magenta Dubois (same technique as the default, rarer coloring)

The direct analog of the recommended Red-Cyan default — a fixed least-squares-fit cross-talk matrix, one matrix multiply, no tunable knobs — but for green/magenta glasses, which are far less common than red/cyan. It's deferred here purely because of that rarity; the code is the same shape as `output3d` §4's RC matrix, just different constants.

```hlsl
if (mode == kAnaGMDubois) {
    // Dubois least-squares-fit cross-talk matrix, green/magenta glasses.
    float r = saturate(-0.062*cA.r - 0.158*cA.g - 0.039*cA.b + 0.529*cB.r + 0.705*cB.g + 0.024*cB.b);
    float g = saturate( 0.284*cA.r + 0.668*cA.g + 0.143*cA.b - 0.016*cB.r - 0.015*cB.g + 0.065*cB.b);
    float b = saturate(-0.015*cA.r - 0.027*cA.g + 0.021*cA.b + 0.009*cB.r + 0.075*cB.g + 0.937*cB.b);
    return float4(FromNorm(float3(r, g, b)), 1.0);
}
```

### 3.2 Simple channel-swap (cheapest, most ghosting)

```hlsl
if (mode == kAnaRC) {   // "3DToElse"-style red/cyan. Cheapest, ghosts badly on saturated color.
    return float4(FromNorm(float3(cA.r, cB.g, cB.b)), 1.0);
}
if (mode == kAnaGM) {   // green/magenta channel assignment
    return float4(FromNorm(float3(cB.r, cA.g, cB.b)), 1.0);
}
```

### 3.3 Half-color compromise (less ghosting than simple, less color loss than Dubois)

```hlsl
if (mode == kAnaRCCompromise) {
    // Per-channel dot products, one eye per output channel group.
    float r = dot(float3( 0.439,  0.447,  0.148), cA.rgb);
    float g = dot(float3( 0.095,  0.934, -0.005), cB.rgb);
    float b = dot(float3(-0.018, -0.028,  1.057), cB.rgb);
    return float4(FromNorm(saturate(float3(r, g, b))), 1.0);
}
```

### 3.4 iaian7 deghosted/contrast (RC / GM / Blue-Amber — two live-tunable knobs)

A different *shape* of algorithm from Dubois: accumulate each output channel as a CONTRAST-weighted sum of one eye's RGB (not a fixed linear matrix), then apply a small cross-channel DEGHOST correction matrix as a second pass. The two knobs (`Contrast`, `DeGhost`) are independently tunable per variant — which the fixed Dubois matrix can't offer, and the whole reason to expose these at all.

```hlsl
if (mode == kAnaRCDeghosted) {
    float contrast = 0.5 + 0.5 * /*Contrast=*/1.0;
    float LOne = contrast * 0.45, ROne = contrast;
    float4 image;
    float4 accum = saturate(cA * float4(LOne, (1-LOne)*0.5, (1-LOne)*0.5, 1));
    image.r = pow(accum.r + accum.g + accum.b, 1.00);
    accum = saturate(cB * float4(1-ROne, ROne, 0, 1));
    image.g = pow(accum.r + accum.g + accum.b, 1.15);
    accum = saturate(cB * float4(1-ROne, 0, ROne, 1));
    image.b = pow(accum.r + accum.g + accum.b, 1.15);

    float DeGhost = 0.006; // 0.06 * 0.1
    float4 acc2 = image;
    image.r = acc2.r + acc2.r*DeGhost         + acc2.g*(-0.5*DeGhost)  + acc2.b*(-0.5*DeGhost);
    image.g = acc2.g + acc2.r*(-0.25*DeGhost) + acc2.g*(0.5*DeGhost)   + acc2.b*(-0.25*DeGhost);
    image.b = acc2.b + acc2.r*(-0.25*DeGhost) + acc2.g*(-0.25*DeGhost) + acc2.b*(0.5*DeGhost);
    return float4(FromNorm(image.rgb), 1.0);
}
// kAnaGMDeghosted, kAnaBlueAmber: same SHAPE (weighted-accumulate per channel, then a
// small cross-channel deghost correction), different channel assignment / contrast
// constants / DeGhost scale. Replicate the pattern, not just one matrix — see
// UU3D's Flat3DShaders.hpp (or VRto3D, which uses the same constants) for the
// exact per-variant constants:
//   https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DShaders.hpp
//   https://github.com/oneup03/VRto3D
```

### 3.5 Why keep multiple techniques at all

They trade off differently. Simple channel-swap is free but ghosts badly on saturated colors. **Dubois** (a fixed least-squares-fit matrix) is the standard "good" choice and cheap — one matrix multiply; Red-Cyan Dubois is the default in `output3d` §4, and Green-Magenta Dubois (§3.1) is the same technique for the rarer coloring. The deghosted/contrast technique adds the two live-tunable knobs so a user can fight a *specific* pair of glasses' actual filter leakage instead of being stuck with Dubois's one-size-fits-all fit — worth the extra ALU only if you expose those knobs in a settings UI. RC/GM/Blue-Amber channel triples cover the three glasses colorings in the wild. If you're *not* exposing knobs, don't need green/magenta, and don't need Blue-Amber, you don't need this section at all — Red-Cyan Dubois alone covers the common case.

### 3.6 The shared-overlay black-screen guard applies here too

If you use `output3d` §6.1's shared-shader overlay architecture, **these branches hit the same anaglyph+UI black-screen bug** as Dubois — worse, actually, because §3.2/§3.3/§3.4 each construct their own hardcoded-`alpha=1.0` `float4`, so a transparent UI pixel comes out opaque (black for the matrix/accumulate paths). Apply the identical fix: check `useUIOverlayMode` first and short-circuit to a plain single-eye sample before any of the math above. Or use the dedicated overlay shader (§6.2) and the whole bug class is gone. See `output3d` §6.1 for the full debugging recipe.

---

## What this skill does NOT cover

- **The shared compose/repack shader, `SampleEye()`, `eye_swap`, the per-eye texture cache, the `Anaglyph()` dispatch + HDR normalize wrapper, the recommended Red-Cyan Dubois anaglyph mode, the core modes (SbS/TaB/interlaced/checkerboard/Katanga/LeiaSR), full-SbS auto-detection, DPI awareness, and the upscale-then-pattern rule** — all in **`output3d`**. This skill only adds the deferred paths (frame-packed, dual-display) and the opt-in anaglyph variants on top of that.
- **The off-axis projection shear, auto-convergence, per-eye pixel-shift math** — see `dynamic3d`.
- **LeiaSR weaver setup / delay-loading** — see `leiasr-integration`.
