---
name: output3d
description: Add stereoscopic 3D output to a real-time renderer, engine/API-agnostic (D3D11/D3D12/Vulkan/OpenGL — examples in HLSL, ports trivially). Covers the two-pass per-eye render loop with a persistent per-eye texture cache (for AFR / native-stereo-fix titles), one shared fullscreen compose/repack shader driving 8 output modes (SbS, TaB, Row/Column Interlaced, Checkerboard, an external-interop passthrough mode for shared-texture stereo consumers like Katanga/VRScreenCap, LeiaSR autostereoscopic, and Red-Cyan Dubois anaglyph — seven further niche anaglyph variants (incl. green-magenta Dubois) and the frame-packed/dual-display output paths are deferred to the sibling `extraoutput3d` skill), the universal upscale-to-output-native rule (every mode bilinear-samples the per-eye texture up to display-native resolution BEFORE selecting an interlace/checkerboard pattern or handing off to the LeiaSR weaver, so those stay display-pixel-exact at any render resolution), per-monitor-DPI-awareness as the fix that keeps the backbuffer in physical pixels (without it the OS silently stretches your output and breaks pixel-exactness), automatic full-SbS (~32:9 ultrawide) native-render-plus-upscale detection with no user toggle, HDR-aware anaglyph (PQ/scRGB normalize-around-the-matrix, and why 10-bit backbuffers aren't automatically HDR), the symmetric-projection compose-time convergence fallback for renderers that can't shear their projection matrix, ghost/crosstalk reduction as two compose-time range-compression levers shipped in every output mode - a contrast squeeze toward mid-grey and a black-floor lift ("foot-room"), the latter targeting the bottom-end clipping that cancelling displays suffer - with the linear-vs-encoded space rule that decides whether they help or hurt, and the two failed refinements you should not rebuild (localizing either lever spatially cannot work because ghosting IS inter-eye difference, so a per-eye correction ghosts in turn; adapting them over time from frame content is structurally sound but visibly pumps the image), the classic anaglyph+UI-overlay alpha-blend bug (and two architectures — shared vs. dedicated overlay shader — that avoid it), and the LeiaSR weaver hand-off. Pairs with the `dynamic3d` skill (clip-space stereo math, auto-convergence, dynamic crosshair) which owns everything upstream of the eye textures this skill composes.
---

# output3d — stereoscopic 3D output modes

This skill is the **output/compositing layer**: given two already-rendered per-eye color textures (and, for depth-aware modes, a depth buffer), how do you get a correctly-formatted stereo-3D image onto the display — Side-by-Side, interlaced, checkerboard, dual-monitor, a shared-texture handoff to an external stereo consumer (Katanga/VRScreenCap-style tools), an autostereoscopic lenticular panel, or any of eight anaglyph variants — while staying HDR-correct, ultrawide-correct, and compatible with alternate-frame-rendering (AFR) engines that only render one eye per frame.

It assumes you already have:
- A per-frame render path that can produce two per-eye color targets (a genuine two-pass stereo render, **or** a single-pass render you intend to feed through the same output pipeline — see §9 for that variant).
- Pipeline/PSO (or GLSL program / Vulkan pipeline) management sufficient to stand up a new fullscreen-triangle pass.
- The off-axis projection shear, auto-convergence, and per-eye pixel-shift math already implemented — that's a separate concern. **See the `dynamic3d` skill.** This skill starts from "I have two eye textures with the correct disparity baked in" (or, for §3.3, "I have two eye textures with NO shear and a CPU-computed shift/scale to apply here instead").

Everything below is written in HLSL against a proven, shipped implementation (a UE-injection stereo mod supporting all these modes concurrently). Porting to GLSL/MSL is mechanical — the only truly API-specific parts are the descriptor/root-signature plumbing in §2 and the resource-barrier sequencing in §2.2, called out separately from the portable shader logic.

**Public reference implementations** (read these when the prose here isn't enough):

- **[UU3D](https://github.com/oneup03/UU3D)** (Unreal Universal 3D Mod, a UEVR fork) — the implementation this skill is drawn from. [`src/mods/vr/flat3d/Flat3DShaders.hpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DShaders.hpp) holds the compose/repack shader and mode enum; [`Flat3DCompositorD3D12.cpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DCompositorD3D12.cpp) / [`Flat3DCompositorD3D11.cpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DCompositorD3D11.cpp) hold the per-eye cache, compose passes, and LeiaSR hand-off; [`Flat3DKatanga.cpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DKatanga.cpp) is the external-interop path (§3.2).
- **[rt64-3D](https://github.com/oneup03/rt64-3D)** (`Stereo3D` branch) — the same compose design in a standalone D3D12/Vulkan renderer: [`src/shaders/StereoComposePS.hlsl`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/shaders/StereoComposePS.hlsl) and [`src/render/rt64_stereo_renderer.cpp`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/render/rt64_stereo_renderer.cpp).
- **[VRto3D](https://github.com/oneup03/VRto3D)** — an OpenVR driver with the same output-mode family and anaglyph constants, under [`vrto3d/src/presenter/`](https://github.com/oneup03/VRto3D/tree/main/vrto3d/src/presenter).

---

## What this delivers

| Family | Mode | How it looks | What the compose shader does |
|---|---|---|---|
| Resampling | `SideBySide` | Left half = L, right half = R | `uv.x < 0.5 → L` else `R`; remap to per-eye `[0,1]` |
| Resampling | `TopAndBottom` | Top half = L, bottom half = R | `uv.y < 0.5 → L` else `R`; remap to per-eye `[0,1]` |
| Line-select | `RowInterlaced` | Even output rows = L, odd = R | `useRight = (outputPixel.y & 1)` |
| Line-select | `ColumnInterlaced` | Even output cols = L, odd = R | `useRight = (outputPixel.x & 1)` |
| Line-select | `Checkerboard` | 1×1 checker per eye | `useRight = (outputPixel.x + outputPixel.y) & 1` |
| Interop | **`Katanga`** — external passthrough (Katanga.exe/VRScreenCap-style consumers) | Local window shows a mono preview; the stereo pair goes out via a shared/interop texture | Local swap chain gets `SampleEye(left)` only; the real SbS composite is published through the interop channel every frame |
| Autostereo | `LeiaSR` | Lenticular, no glasses | Compose SbS into an intermediate, hand off to the SR weaver (§7) |
| Anaglyph (Dubois) | `AnaRCDubois` | Red/cyan glasses | Dubois least-squares cross-talk matrix (§4); recommended default. Seven other anaglyph variants (incl. green-magenta Dubois) are deferred to the `extraoutput3d` skill |

7 of 8 modes (everything but LeiaSR, which needs the weaver) share **one fullscreen compose shader**, selected by a `mode` constant — one PSO for the world/scene pass, one for premultiplied-alpha UI overlay (or, per §6, a second dedicated overlay shader entirely).

---

## Architecture at a glance

```
 per-eye render (2x, or 1x for AFR — see below)
        │
        ▼
 ┌─────────────────────────────┐     ┌──────────────────────────────┐
 │ persistent LEFT eye texture │     │ persistent RIGHT eye texture │
 │ (refreshed when the left    │     │ (refreshed when the right    │
 │  eye rendered this frame)   │     │  eye rendered this frame)    │
 └─────────────────────────────┘     └──────────────────────────────┘
        │                                       │
        └────────────────┬──────────────────────┘
                          ▼
              ┌───────────────────────────┐
              │ compose/repack pass        │   mode-keyed switch, §3
              │  → real swap chain, OR     │
              │  → SbS intermediate → SR   │   LeiaSR only, §7
              │    weaver → swap chain     │
              └───────────────────────────┘
                          │
                          ▼
              overlay pass(es): UI, crosshair,
              menu, cursor — §6
```

The persistent per-eye textures (not a direct read of whatever the engine rendered this frame) are what makes AFR and "native-stereo-fix"-style engines work: on a full-stereo frame both textures refresh; on an AFR frame (the engine renders exactly one eye) only that eye's texture is copied, and the compose pass always weaves the freshest available pair — one eye one frame stale, same visual contract as any AFR HMD path. Never special-case "AFR is on, skip stereo" — just refresh fewer textures per frame.

---

## 1. What arrives at this layer

Two possibilities, both valid inputs to this skill's compose pass:

- **True per-eye shear already applied.** The two eye textures were rendered through off-axis projections with the convergence shear baked in (see `dynamic3d` §1.1). The compose pass in §3 just needs to sample and format them — no further depth math here.
- **No shear available** (symmetric-only renderer, or a wrapper/injection context that doesn't own the projection matrix). Render both eyes with identical, un-sheared projections, and let the compose pass itself apply a whole-image horizontal shift + zoom-to-cover per eye (§3.3) — this is `dynamic3d` §5.3's fallback, and the shader-side half of it lives here.

Either way, by the time you reach this skill, you have two eye textures (plus, for the second case, a `scene_shift`/`scene_scale` pair computed upstream) and the rest is display formatting.

A third shape, from single-pass Vulkan multiview: **one 2-layer array image** (layer 0 = left, layer 1 = right) instead of two textures. Bind it as a `sampler2DArray` / `Texture2DArray` and let `SampleEye()` pick the layer — every mode below works unchanged, with no per-eye copies. How that image gets produced (and the many operations that silently touch only layer 0) is in `vulkan-multiview-stereo`.

---

## 2. The per-eye texture cache

```cpp
// Two persistent, render-target-capable textures at eye resolution — NOT
// aliases into whatever the engine's current frame target is. Recreate only
// on resolution/format change.
Texture2D eyeTex[2];  // [0] = left, [1] = right

void refreshEyeCache(bool afrFrame, bool afrLeftIsFresh, bool nativeStereoFixLayout) {
    // Native-stereo-fix titles: the engine only ever writes the LEFT half of
    // its own double-wide/side-by-side render target; the right eye comes
    // from wherever your engine's scene-capture/secondary-view path lands —
    // same source a stock HMD submit path would consume.
    bool leftHalfIsSource = nativeStereoFixLayout || afrFrame;

    if (afrFrame) {
        int fresh = afrLeftIsFresh ? 0 : 1;
        copyEyeHalf(fresh, leftHalfIsSource);
        // The OTHER eye texture is untouched this frame — still valid from
        // last frame. Transition it straight to shader-readable; don't wait
        // on a copy that isn't happening.
    } else {
        copyEyeHalf(0, leftHalfIsSource);
        copyEyeHalf(1, leftHalfIsSource);
    }
}
```

`copyEyeHalf` is a plain `CopyTextureRegion`/`glCopyImageSubData`/equivalent from whichever half of the engine's render target this eye's data actually lives in this frame — a source-box selection problem, not a shader problem. Two box-selection wrinkles worth encoding explicitly rather than assuming symmetry:

- **AFR always writes the same physical half** regardless of which eye it logically is — the engine doesn't know "this is the right eye," it just renders into "the" target. Key the copy's *destination* eye texture off the frame's role flag, and its *source box* off `leftHalfIsSource`, independently.
- **`eye_swap`** (a user-facing "my glasses/panel are backwards" toggle) is applied at the *sampling* end in §3, not here — keep the cache copy dumb and physical, do all the "which eye is which" logic in one place downstream.

### 2.1 Descriptor/root-signature plumbing (D3D12-flavored; adapt per API)

```cpp
RangeBuilder b;
b.addPushConstant(/*space=*/0, /*reg=*/0, sizeof(RepackConstants), PIXEL);
b.addTexture(/*reg=*/0);   // eye_left
b.addTexture(/*reg=*/1);   // eye_right
b.addImmutableSampler(/*reg=*/0, linear_clamp_sampler);
```

Two PSOs from one shader/layout: `Copy` blend for the world/scene compose (overwrites the destination), `AlphaBlend` for any overlay pass that reuses this shader (§6 covers whether that's a good idea). Use a vertexless fullscreen triangle (`SV_VertexID`-driven positions) for both — no VB, no IA state, fewer barriers than a quad.

---

## 3. The compose/repack shader (one shader, 7 modes)

```hlsl
cbuffer RepackParams : register(b0) {
    int2  out_size;      // swap-chain / output pixel dimensions
    int   mode;
    int   eye_swap;
    int   colorspace;    // OUTPUT space: 0 SDR, 1 HDR10/PQ, 2 scRGB — see §5
    float paper_white;   // nits, HDR only
    int   src_srgb;       // eye textures are 8-bit sRGB under an HDR output — see §5
    // Symmetric-projection convergence fallback (§1, §3.3) — identity (0, 1)
    // when the renderer applies true per-eye shear instead.
    float scene_shift_uv; // LEFT-eye image shift, eye-UV units (right eye = -this)
    float scene_scale;    // >= 1: zoom so the shifted image still fills the eye
};

Texture2D eye_left  : register(t0);
Texture2D eye_right : register(t1);
SamplerState samp   : register(s0);

static const int kSbS = 0, kTaB = 1, kRowInterlaced = 2, kColInterlaced = 3,
                  kCheckerboard = 4, kLeiaSR = 5,
                  kKatanga = 6,   // external interop passthrough — see §3.2
                  kAnaRCDubois = 7;
                  // Other anaglyph modes (green-magenta Dubois, simple, deghosted,
                  // compromise, blue-amber) are extra branches on Anaglyph() —
                  // see extraoutput3d §3.

// half_idx: 0 = left half of the OUTPUT layout, 1 = right/bottom half.
// This is the ONE place eye_swap and the symmetric-mode compose-time shift
// both apply — every mode below routes through here.
float4 SampleEye(int half_idx, float u, float v) {
    int eye = (eye_swap != 0) ? (1 - half_idx) : half_idx;

    // Symmetric-projection convergence (§3.3): equivalent to the projection's
    // [2][0] shear, applied here instead as a per-eye horizontal UV shift +
    // zoom so the shifted image still covers the frame.
    float dir = (eye == 0) ? 1.0 : -1.0;
    float2 uv = float2((u - 0.5 - dir * scene_shift_uv) / scene_scale + 0.5,
                       (v - 0.5) / scene_scale + 0.5);

    return (eye == 0) ? eye_left.Sample(samp, uv) : eye_right.Sample(samp, uv);
}

float4 Repack(float2 uv, float2 outputPixel) {
    if (mode == kSbS || mode == kLeiaSR) {
        int   half_idx = (uv.x < 0.5) ? 0 : 1;
        float u_half   = (uv.x < 0.5) ? (uv.x * 2.0) : ((uv.x - 0.5) * 2.0);
        return SampleEye(half_idx, u_half, uv.y);
    }

    if (mode == kKatanga) {
        // Local window preview only (mono); the actual stereo pair leaves via
        // whatever interop channel your consumer expects (§3.2).
        return SampleEye(0, uv.x, uv.y);
    }

    if (mode == kTaB) {
        // Top rows = left eye, bottom rows = right eye, no gap.
        if (uv.y < 0.5) return SampleEye(0, uv.x, uv.y * 2.0);
        return SampleEye(1, uv.x, (uv.y - 0.5) * 2.0);
    }

    // §3.1 — pattern selected from the OUTPUT pixel coordinate, AFTER the
    // (possibly-upscaling) per-eye sample above. Display-pixel-exact at any
    // render resolution.
    if (mode == kRowInterlaced)  return SampleEye((int)outputPixel.y & 1, uv.x, uv.y);
    if (mode == kColInterlaced)  return SampleEye((int)outputPixel.x & 1, uv.x, uv.y);
    if (mode == kCheckerboard)   return SampleEye(((int)outputPixel.x + (int)outputPixel.y) & 1, uv.x, uv.y);

    // §4 — anaglyph family; every variant samples both full eyes.
    return Anaglyph(mode, SampleEye(0, uv.x, uv.y), SampleEye(1, uv.x, uv.y));
}
```

### 3.1 Upscale-to-output-native, in every mode, before anything else

**The eye textures are rendered at the game's render resolution; the compose pass runs at the output (display-native) resolution. The upscale between the two is not a separate pass — it's the bilinear `Sample` inside `SampleEye`.** `SampleEye` samples the (possibly lower-resolution) eye texture at the **output** UV, so the world content is upscaled the instant it's read, in *every* mode. This is deliberate and universal, not an interlace-only trick:

- The per-eye render size is typically a fraction of native (a render-scale slider — Auto/100/75/67/50/33% — for performance), while the swap chain / compose target stays display-native. A linear sampler (`MIN_MAG_MIP_LINEAR`) plus a display-native render target *is* the upscale; no explicit resample pass exists.
- **The upscale must precede the pattern selection AND the LeiaSR weave.** For interlace/checkerboard, only *after* the per-eye sample does `RowInterlaced`/`ColInterlaced`/`Checkerboard` pick a pattern, and it picks it from `outputPixel` (`SV_Position`, i.e. real swap-chain pixel coordinates), never from the eye texture's own texel grid. For LeiaSR, build the SbS weaver-input texture at **display** resolution first, then weave (§7) — upscale-then-weave. Get the order backwards — pattern-then-upscale, or weave a render-resolution SbS — and the pattern pitch / lenticular alignment drifts whenever render resolution ≠ output resolution (dynamic res, supersampling, the render-scale slider), reading as shimmering moiré that changes with render settings instead of a fixed, display-native pattern.

### 3.1.1 …which only works if the backbuffer is truly display-native pixels — so make the process DPI-aware

The upscale-to-output rule assumes your compose target's pixels map 1:1 to physical panel pixels. On a display with OS scaling (150%, 200%, …) that assumption is silently false unless the process is **per-monitor-DPI-aware**: Windows maps a non-DPI-aware process's window — and therefore your backbuffer, and therefore the interlace/checkerboard/LeiaSR pattern — into a **DPI-virtualized sub-region** of the native panel and stretches it up afterward. That stretch happens *after* your last shader; you cannot correct for it. The line pattern softens, the lenticular weave misaligns, and "1080p out" isn't actually 1080 physical rows.

The fix is a process-init action, not a shader change — **declare per-monitor-DPI-awareness before the game creates its window or swap chain**:

```cpp
// As early as possible — DllMain(DLL_PROCESS_ATTACH) for an injected mod, or the
// app manifest for a native app. Must run before the first window/swapchain.
SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);
// (fallbacks: SetProcessDpiAwareness(PROCESS_PER_MONITOR_DPI_AWARE) / SetProcessDPIAware())
```

Two wrinkles worth encoding:
- **Verify it took.** Read it back (`GetAwarenessFromDpiAwarenessContext(GetThreadDpiAwarenessContext())`) and log per-monitor / system / unaware, so you can confirm injection landed early enough. If a backbuffer-vs-display size mismatch survives anyway (`GetContainingOutput` or platform equivalent), warn rather than silently shipping a soft pattern.
- **You may still want the *engine* to think DPI is 100%.** Making the process DPI-aware is for *your* physical-pixel output; a UE-style engine that then queries `GetDpiForWindow`/`GetDpiForSystem`/`GetDpiForMonitor` may scale its own UI by the real DPI. If that's unwanted, inline-hook those three to return 96 (scale 1.0) for the engine while your own output path keeps the true DPI — process-aware for your pixels, spoofed-96 for the engine's layout.

### 3.2 Katanga / external-interop passthrough

Some 3D output paths aren't "draw it on my own swap chain" at all — a separate consumer process (Katanga.exe, VRScreenCap, a 3D-Vision-style driver shim, a screen-capture-based stereo tool) wants the composed SbS/TaB pair delivered through a shared/interop channel (a named shared texture, a memory-mapped file + mutex rendezvous, a D3D11-on-12 bridge if your renderer and the consumer's expected API differ) every frame, while your own window keeps presenting a normal mono preview so the game remains usable without the external tool running.

Two things this mode needs that the others don't:
- **Force the game into windowed/bordered-window mode.** These consumers typically composite from a normal desktop window (capture- or shared-surface-based); an exclusive-fullscreen swap chain is invisible to them. If the engine has its own "apply resolution" pipeline, prefer that lever first and fall back to directly clearing `WS_POPUP`/setting `WS_OVERLAPPEDWINDOW` only if the engine ignores the request after a few frames — don't fight a title that already went windowed correctly.
- **Publish via the SAME format/resolution contract the consumer expects**, independent of what your local preview renders — the interop surface's format (`view_format_for` — TYPELESS eye formats need a concrete UNORM/FLOAT view to be sampled/copied cross-API) and the rendezvous names/handles (e.g. Katanga's well-known mapped-file and mutex names) are the consumer's contract, not yours to redefine.

### 3.3 Symmetric-projection convergence fallback

When §1's "no shear available" case applies, `scene_shift_uv`/`scene_scale` (computed upstream — `dynamic3d` §5.3) get baked directly into `SampleEye`'s per-eye UV remap above. This is the compose-shader half of that fallback: every mode in this file's `Repack` routes eye sampling through `SampleEye`, so enabling the fallback is a single upstream flag flip, not a per-mode shader change. `scene_scale` here is deliberately the *cover* strategy (zoom in to hide the revealed edge) — see `dynamic3d` §5.2 for why this is the one case where cover, not fit, is correct: there is no other content behind a whole-eye shift, so cropping a border of real image is strictly better than exposing undefined edge pixels.

### 3.4 Ghost reduction — two levers, both global

Ghosting (a faint copy of the other eye bleeding through) is the most common "the 3D looks wrong" complaint, and the cheapest lever against it lives right here in the compose shader.

Every stereo display leaks some of each eye's image into the other. How visible that leak is depends on the **brightness difference between the eyes**, so compressing the signal range before it reaches the display reduces what you see — the standard range-compression approach from the stereo-crosstalk literature. Some displays (autostereo panels in particular) additionally **cancel** crosstalk themselves, pre-subtracting a fraction of the opposite eye. That inversion drives values past the ends of the range, where the render target clamps them, and **the clamped part is what survives as a ghost**. So on a cancelling display the correction is already running and residual ghosting means it clipped, not that it's absent.

Apply **last**, after any colour grading, so what gets compressed is what actually reaches the display:

```hlsl
// contrast: 1.0 = off.  lift: 0.0 = off.  Both exact no-ops at their defaults.
float3 GhostReduce(float3 c, float contrast, float lift) {
    float3 lin = pow(saturate(c), 2.2);
    lin = (lin - 0.5) * contrast + 0.5;   // squeeze toward mid-grey
    lin = lin * (1.0 - lift) + lift;      // raise the black floor
    return pow(saturate(lin), 1.0 / 2.2);
}
```

- **contrast** — shrinks `|L − R|` directly, and leaves `(1 − contrast)/2` of headroom at *each* end of the range. Costs contrast across the whole image. Try `0.90` first, lower only if edges still ghost.
- **lift** — raises the black floor and leaves white alone. Cancellation clips at the **bottom** (subtracting the opposite eye drives dark pixels below zero), so lift targets that specifically while contrast spends most of its effect squeezing highlights that were never the problem. The literature calls the resulting margin **foot-room**. Costs black level rather than contrast. Try `0.02`–`0.05`; blacks go grey fast above that.

Ship both and let the user pick — which wins is panel- and content-dependent. **Lift only helps on displays that actually cancel**; where nothing subtracts there is no clipping to relieve and only contrast reduces visible ghosting.

Gate to SDR (the remap is defined on sRGB-encoded `[0,1]`, so skip it under PQ/scRGB exactly as §5 does), and exclude screenshot / external-interop captures (§3.2) — a capture shouldn't bake in a fix for one display.

> ⚠ **The space rule is not optional, and getting it backwards makes things worse.** The remap must run in whatever space the display's cancellation runs in. Cancelling displays generally work in **linear light**, which is what the code above assumes — pivot `0.5` in linear, via a plain `pow(2.2)` rather than the piecewise sRGB curve, so the two stay consistent. If your output path hands the display already-encoded values instead, drop both `pow()` calls and pivot around `0.5` in your own encoding. On LeiaSR specifically this is decided by your `setShaderSRGBConversion(read, …)` choice — see `leiasr-integration` §A.11.

#### Do NOT try to make this smart

Both obvious improvements have been built and both failed. They fail for the same structural reason, so don't spend the time:

**Localizing it (spatially) cannot work.** The intuition — "only compress where the eyes actually differ, keep full contrast elsewhere" — is exactly what the papers recommend, and it's wrong here. **Ghosting *is* inter-eye difference**, so any correction applied to one eye and not the other *adds* inter-eye difference and leaks in turn:

- A localized lift leaves a grey patch that only stops the ghost once it has grown to cover everything — i.e. once it's global again. Globally it worked through whole-frame adaptation; a patch can't reproduce that, because the un-lifted surroundings stay as a reference.
- Dimming the leak at its *source* instead (attenuate the bright eye where the fellow eye is dark) carves a dark rim into one eye — and that rim is new inter-eye difference, which ghosts against bright backgrounds. You trade a bright ghost for a dark one.

A risk map that correctly identifies the ghosting regions does not rescue either approach; the problem isn't finding the pixels, it's that a per-eye correction is itself ghostable.

**Adapting it over time is visible.** Deriving one *global* scalar per frame from frame content is structurally sound — it applies identically to both eyes at every pixel, so it can't manufacture difference — and the measurement is cheap (reduce both eyes to a single number via a 1×1 target with non-blocking readback; use the **fraction** of sampled points holding high-contrast inter-eye difference, never a maximum, because in ordinary 3D content the eyes differ almost everywhere from disparity alone and a maximum saturates immediately). But the result is that the whole image visibly brightens and darkens as content changes. Attack-fast/release-slow with a deadband reduces it; slowing the release further just trades pumping for the correction arriving too late to matter. **A fixed slider value is less distracting than a correct but moving one.**

#### Implementation gotchas

- **Constant-buffer packing.** Adding these floats shifts your compose cbuffer. HLSL rounds a cbuffer up to a 16-byte multiple, and D3D11 requires `ByteWidth` to be a 16-byte multiple — if you pass `sizeof(struct)` straight in, an unpadded struct fails `CreateBuffer`. Prefer consuming existing tail padding, add explicit padding otherwise, and verify against `fxc /dumpbin` reflection rather than counting by hand. On D3D12 root constants, update the dword count *and* any `static_assert` on the struct size.
- **If your shader is a C++ raw string literal, MSVC caps one literal at 16380 bytes.** Adding a documented helper can push a large compose shader over. Split it — adjacent string literals concatenate: `)" /* split */ R"(` at a function boundary — and keep long rationale in C++ comments outside the literal, where it costs no budget.
- **Default to exactly the no-op values** (`1.0` / `0.0`) and branch on them, so the untouched path stays bit-exact and free.

---

## 4. Anaglyph — Red-Cyan Dubois (the recommended variant)

One anaglyph mode ships in this skill's default set: **Red-Cyan Dubois** — a least-squares-fit cross-talk matrix that's the standard "good" choice and cheap (one matrix multiply, no tunable knobs). Red/cyan is overwhelmingly the glasses coloring people actually own, so it's the only anaglyph mode worth putting in front of a user by default. **Recommendation: use Red-Cyan Dubois unless a user explicitly asks for another anaglyph style.**

```hlsl
// cA = left eye sample, cB = right eye sample, already normalized to ~[0,1]
// display-referred space (§5 — identity in SDR). FromNorm re-encodes for HDR.
float4 Anaglyph(int mode, float4 cA, float4 cB) {
    if (mode == kAnaRCDubois) {
        // Dubois least-squares-fit cross-talk matrix, red/cyan glasses.
        float r = saturate( 0.437*cA.r + 0.449*cA.g + 0.164*cA.b - 0.011*cB.r - 0.032*cB.g - 0.007*cB.b);
        float g = saturate(-0.062*cA.r - 0.062*cA.g - 0.024*cA.b + 0.377*cB.r + 0.761*cB.g + 0.009*cB.b);
        float b = saturate(-0.048*cA.r - 0.050*cA.g - 0.017*cA.b - 0.026*cB.r - 0.093*cB.g + 1.234*cB.b);
        return float4(FromNorm(float3(r, g, b)), 1.0);
    }

    return float4(cA.rgb, 1.0); // unreachable if mode is validated upstream
}
```

**Seven other anaglyph variants live in the `extraoutput3d` skill (§3):** Green-Magenta Dubois (the same technique for the rarer green/magenta coloring), simple channel-swap (RC/GM), iaian7 deghosted/contrast (RC/GM/Blue-Amber, with two live-tunable knobs), and half-color compromise (RC). They're pure extra branches on this same `Anaglyph()` dispatch (same `ToNorm`/`FromNorm` HDR wrapper, §5), separated out only because Red-Cyan Dubois is the right default. **Don't add them unless a user explicitly asks** — the extras exist for the green/magenta or Blue-Amber colorings, or for tuning against a specific pair of glasses' filter leakage.

---

## 5. HDR-aware anaglyph (and pass-through modes)

The 7 non-anaglyph modes (§3) are pure UV remaps — they pass HDR pixels through untouched and work natively in any format/color space, **except** when the eye textures are 8-bit sRGB but the output swap chain is HDR (common when the engine renders SDR but you want an HDR-capable output path) — that specific combination still needs a `to_normalized`/`from_normalized` re-encode even for a "just resample" mode, or SDR eye content displays either washed-out or crushed on an HDR panel.

The anaglyph matrices, however, are **defined for display-referred SDR primaries** — running them directly on PQ or scRGB-encoded values produces nonsense (the matrices assume roughly-perceptual [0,1] values, not PQ codewords or linear-light nits/80). Wrap every anaglyph variant in normalize-in / re-encode-out:

```hlsl
float3 pq_to_nits(float3 e) { /* ST-2084 inverse EOTF */ }
float3 nits_to_pq(float3 n) { /* ST-2084 EOTF */ }

float3 ToNorm(float3 c) {
    if (src_srgb != 0 || colorspace == 0) return c;              // already sRGB [0,1]
    if (colorspace == 1) return saturate(LinearToSrgb(pq_to_nits(c) / paper_white));   // PQ
    return saturate(LinearToSrgb(c * (80.0 / paper_white)));                            // scRGB: 1.0 = 80 nits
}

float3 FromNorm(float3 c) {
    if (colorspace == 1) return nits_to_pq(SrgbToLinear(c) * paper_white);
    if (colorspace == 2) return SrgbToLinear(c) * (paper_white / 80.0);
    return c;
}
```

Sample each eye, `ToNorm()` before the matrix, run the matrix in normalized space exactly as in §4, `FromNorm()` the result. `paper_white` is a user-exposed nits value (default ~200) that calibrates how "1.0" in the SDR-defined matrix maps into the HDR output's absolute brightness — without it, HDR anaglyph output is either too dim or blown out depending on content. Document this path as *approximate*: the anaglyph matrices are calibrated against real glasses' spectral filters under SDR viewing conditions, and HDR peak brightness doesn't change what wavelength a red-lens filter blocks — accuracy claims stop at "reasonable," not "correct."

### The 10-bit-isn't-automatically-HDR trap

`R10G10B10A2` swap-chain formats are used by both plain SDR titles (10-bit color depth, standard gamma) and HDR10 titles (10-bit + PQ transfer function) — the pixel format alone does not tell you which. Treat a 10-bit backbuffer as SDR unless the swap chain has **explicitly** signaled an HDR color space (`IDXGISwapChain3::GetColorSpace1` / `SetColorSpace1` observed as PQ, or the platform equivalent). Assuming "10-bit ⇒ HDR ⇒ apply PQ decode" on a title that's actually SDR-with-10-bit-precision washes the image out — this is a real, easy-to-hit bug, not a hypothetical.

```cpp
switch (backbuffer_format) {
case R16G16B16A16_FLOAT: colorspace = SCRGB; break;
case R10G10B10A2_UNORM:
    colorspace = swapchain_signaled_pq ? HDR10_PQ : SDR;   // NOT format-alone
    break;
default: colorspace = SDR; break;
}
```

`R16G16B16A16_FLOAT` has no such ambiguity in practice — treat it as scRGB linear unconditionally.

---

## 6. UI/overlay compositing — two architectures

A UI/HUD layer needs its own composite pass on top of the world/scene compose, typically alpha-blended so translucent pixels show the 3D scene through. Two architectures both work; pick based on how much you want to reuse §3's shader.

### 6.1 Shared shader (reuse the world compose shader for overlays too)

Bind the UI texture to *both* eye slots, set `useUIOverlayMode`, switch the blend state to AlphaBlend, and run the exact same `Repack`/mode switch. Cheap to add (no new shader), and for SbS/TaB/interlaced/checkerboard it's correct for free — the UI ends up sampled identically regardless of which eye's "half" a given output pixel maps to, so it reads as sitting at the screen plane (zero parallax) automatically.

**The trap: anaglyph.** If the anaglyph branch also runs under `useUIOverlayMode`, it samples the UI texture through both eye slots, runs it through the color matrix, and returns `float4(matrixMixedRGB, 1.0)`. An `AlphaBlend` pipeline reads `alpha = 1.0` as "fully replace the destination" — and `matrixMixedRGB` for any UI pixel that's actually *transparent* (alpha 0 in the source) comes out as black (matrix of all-zero input is zero). Every transparent UI pixel becomes opaque black, and the world compose underneath — which was correct — gets painted over. Symptom: world renders fine, then the UI overlay pass paints the whole swap chain black (or black from frame 1, if the UI is present immediately).

**Fix:** inside the anaglyph branch, check `useUIOverlayMode` first and short-circuit to a plain single-eye sample (which carries the UI texture's real alpha) before ever reaching the matrix math. Every other mode is naturally immune because they all bottom out in a plain `SampleEye` call that already respects the overlay-mode alpha path — only the anaglyph branch constructs its own hardcoded-alpha `float4`, which is what introduces the bug.

Debugging recipe if you hit an unexplained black screen in anaglyph-only: replace the anaglyph return with a solid color (confirms the shader runs) → replace with a single plain `SampleEye` (confirms sampling works) → mix both eyes with a hardcoded `alpha = 1.0` (reproduces the bug) → that's the smoking gun; add the `useUIOverlayMode` bypass.

### 6.2 Dedicated overlay shader (structurally avoids the bug)

Alternative, used in the shipped implementation this skill is drawn from ([UU3D](https://github.com/oneup03/UU3D), which also carries the DIBR extras this pass leaves out): give overlays their **own** shader/PSO entirely, decoupled from `Repack`'s mode switch. It never has an anaglyph branch to guard, because it never runs the world-compose shader at all — it always does a straightforward "sample source texture, respect its real alpha, write premultiplied" regardless of which output mode is active. The overlay pass instead needs to know each eye's own per-pixel disparity/shift (from `dynamic3d`) to place UI at the right screen-plane depth per eye, which is orthogonal to which *output mode* is selected.

Trade-off: one more shader/PSO to maintain, but §6.1's entire bug class doesn't exist by construction, and it decouples "how do overlays get positioned in depth" from "how does the world get formatted for the display" — two genuinely separate concerns that §6.1 conflates by sharing one mode switch for both. **Recommended for new projects** with the caveat that you now need to keep the overlay shader's eye-selection/`eye_swap` handling in sync with §3's `SampleEye` by hand, since they're no longer the same code path.

Either architecture: bind the SAME UI texture to both eye slots for the SbS/TaB/interlaced/checkerboard family so the UI is mirrored per-eye at zero parallax (reads as "the UI sits on the screen"); a per-eye-shifted UI (dynamic crosshair/HUD-at-depth) is a `dynamic3d`-driven exception to that default, not the common case.

---

## 7. LeiaSR / lenticular weaver hand-off

LeiaSR (Simulated Reality) autostereo panels take a plain SbS-packed 2D texture and do their own eye-tracked lenticular interleave via a vendor weaver call — your job is producing a correct SbS intermediate (§3's `kSbS`/`kLeiaSR` path already does this) and handing it off correctly:

```cpp
// 1. Compose world stereo as SbS into an INTERMEDIATE render target (not the
//    swap chain directly), sized to the swap chain (desktop-native pixels).
composeWorld(mode=SideBySide, target=leiaSRIntermediate);

// 2. Composite UI INTO THE SAME INTERMEDIATE before the weave (§6) — an SbS
//    UI layer composited AFTER the weave shows as two visible UI copies
//    through the lens, one per eye column.
composeOverlay(target=leiaSRIntermediate);

// 3. Weave: intermediate (SHADER_READ) -> swap chain (bound as RTV).
weaver.setInputViewTexture(intermediateResource, w, h, format);
weaver.setOutputFormat(swapChainNativeFormat);   // query it — never hardcode
weaver.weave();
```

Activation order matters: `SRContext::create()` → `CreateDX##Weaver(...)` → `weaver->setShaderSRGBConversion(...)` → **then** `context->initialize()` (starts eye tracking). Calling `initialize()` before the weaver exists produces a weaver that renders a correct still image but never responds to head movement.

The raw SDK calls above are shown so the hand-off is legible. In practice, don't write them: the `bo3b/SR-lib` submodule's `api_expansion` branch ships an `SRInterfaceDX9/DX11/DX12/OGL` wrapper that does this ordering, the delay-load probing and the exception containment for you, plus a CMake package (`SRLib::SR` + `srlib_apply_delayload`). See `leiasr-integration` A.0.

Residual ghosting on high-contrast content (bright HUD or subtitles over a dark scene) is usually *not* missing anti-crosstalk — the weaver applies it automatically. It's the correction clipping, and the fix is the contrast remap in §3.4, applied to this SbS intermediate before the weave. See `leiasr-integration` A.11.

Shipping a binary that still launches on machines without the SR runtime installed needs `/DELAYLOAD` on every SR DLL plus a `LoadLibraryW` pre-check before any SDK call (the delay-load thunk throws an SEH exception on a missing DLL, which plain `catch(...)` won't catch without `/EHa`, which you don't want ABI-wide). **This entire topic — weaver setup, delay-load, SEH gotchas, HDR tone-mapping since the weaver wants SDR input — has its own skill: `leiasr-integration`.** Use this section only for where LeiaSR plugs into the mode matrix above (§3's `kLeiaSR` producing the SbS source); go there for the rest.

---

## 8. Full-SbS on ultrawide (~32:9) — automatic, not a toggle

On a ~32:9 ultrawide (full-SbS glasses/panels split it into two 16:9 eyes), each output half is *already* a native 16:9 slice — so this is "full SbS," not the "half SbS" case where each eye is squeezed into half of a 16:9 window. The trap: if the engine rendered each eye at the *full* window width (32:9) and you stretch that into a 16:9 half-slice, every eye is squashed horizontally. Full-SbS is therefore **not a separate output mode and not a user toggle** — it's an automatic property you detect from the output aspect and respond to by changing *what resolution you render each eye at*.

### 8.1 Detect it from the output aspect

Each frame, classify the SbS-family modes (any mode that lays the two eyes out as a double-wide left/right surface) as full-SbS when the output is double-wide:

```cpp
const bool sbs_family = (mode == kSbS) /* || any other double-wide L/R layout */;
const bool full_sbs   = sbs_family && swap_h != 0 &&
                        ((float)swap_w / (float)swap_h) >= 2.5f;   // ~32:9 and wider
```

Store it as an atomic the compose/UI path reads; recompute it every frame (the window can move between monitors). On a normal 16:9 / half-SbS panel the ratio test fails, `full_sbs = false`, and none of the rest engages.

### 8.2 Respond with native-render + upscale (the real fix)

The correct response is **not** to crop — it's to make the engine render each eye at its true per-eye (16:9-span) width and then upscale to the full output width in compose:

1. **Feed the engine a half-width client rect.** Inline-hook the `GetClientRect`-family calls the engine sizes its render targets from and halve the reported width, so it lays out each eye at native per-eye width instead of the full 32:9 width. The swap chain and window stay at the **full** native width.
2. **Upscale in compose.** §3.1's universal upscale already does this — each half-slice is bilinear-sampled from the (now correctly-proportioned, half-width) eye texture up to the full half of the output. No squish, because the source is a true 16:9 eye.
3. **Re-drive the render size once if the game booted wide.** If the title already came up at the full 32:9 width, its targets are baked wrong; re-issue the engine's own "apply resolution" once at the half width (e.g. UE `r.SetRes <halfW>x<H>w`) so it re-bakes at native per-eye. Do this once per transition, not every frame.

**If you own the engine, the lever is its aspect-ratio target, not a client-rect hook.** Steps 1 and 3 above are the injection shape of this fix — hooking the call the engine sizes its targets from, because you cannot reach inside it. In a renderer you control, the same fix is one clamp where the render aspect is chosen: when stereo is on, cap the per-eye target at the aspect the eye is *shown* at (16:9) instead of deriving it from the whole panel. Everything downstream — target sizes, resolution scale — follows from that one value.

**Shipping only the crop is a silent 2x cost.** This is the failure mode to watch for, because nothing complains: the crop in §8.3 makes the picture *correct*, so the image looks right, the geometry is right, and the only symptom is that you are rendering two eyes at full panel width and then throwing half of each away. On a 5120x1440 panel that is four times the pixels of a mono 16:9 frame in order to display two. If your renderer has the lever at all, the crop should be a no-op on the path that uses it, not a load-bearing part of the output.

This is strictly better than stretching a 32:9-rendered eye: you render *fewer* pixels per eye (half the width) and each eye is geometrically correct.

### 8.3 Residual: reconciling the UI/HUD target aspect

The world path above fixes the scene. A separately-composited UI/HUD layer (§6) whose target aspect differs from the per-eye slice still needs an explicit reconcile — expose it as a small enum rather than hard-coding one policy, because the right answer is content-dependent:

- **Crop** — center a matching-aspect slice of the UI (lose the edges). Good when the UI has safe margins.
- **Fit** (letterbox/pillarbox) — scale to fit, pad the remainder. Good when no UI element may be lost.
- **Stretch** — fill, accept distortion. Rarely ideal, occasionally wanted.

The centered-aspect crop is the fallback the *pure-compositor* case degrades to when you have no client-rect lever to re-drive the engine's render size (§8.2 unavailable): crop a centered 16:9 slice per eye instead of stretching a 32:9 render.

```cpp
const float texAspect = texW / texH;   // per-eye source slice aspect
constexpr float kTargetEye = 16.0f / 9.0f;
if (texAspect > kTargetEye + 1e-4f) {  // source wider than the eye slot → crop
    const float frac = kTargetEye / texAspect;
    contentExtent = { frac, 1.0f };
    contentOrigin = { (1.0f - frac) * 0.5f, 0.0f };
}
```

All of §8 is a no-op on 16:9 / half-SbS displays (every branch yields identity), so the detection and reconcile can run unconditionally across all modes — harmless where the aspect already matches, necessary where it doesn't.

---

## 9. Advanced/experimental: single-pass depth-image-based-rendering (DIBR) eye synthesis

Everything above assumes a genuine two-pass (or AFR) render producing two real eye images. An alternative worth knowing about, at real cost/maturity trade-offs: render **once** (mono) with a depth buffer, and synthesize the second eye by reprojecting/shifting pixels by a disparity derived from depth, instead of rendering the scene twice. This roughly halves render cost at the expense of synthesis artifacts (disocclusion holes where the synthesized eye reveals geometry the mono pass never rendered).

Sketch of the shape of the technique (**not** a drop-in recipe — this is genuinely the least mature, most content-dependent part of any stereo output pipeline and deserves its own scrutiny per project):

- **Screen-space disparity shift**, the fail-closed default: shift each pixel horizontally by an amount driven by its depth (nearer = larger shift, same sign convention as `dynamic3d`'s `shift_px`), governed by a single user-tunable "disparity pixels at some reference depth" knob rather than a full physical IPD/convergence pair — simpler to reason about when you're synthesizing rather than truly re-rendering.
- **True reprojection** (optional, stronger): instead of a flat per-pixel disparity heuristic, map source clip-space directly into the target eye's clip space with a real projective transform when you have valid per-eye projections and a known baseline — a `reprojection_strength` blend lets you dial between the cheap heuristic and the full solve per title.
- **Edge stabilization**: depth discontinuities (silhouette edges) produce the worst disocclusion artifacts; detecting a depth-edge threshold and damping the shift near it trades some 3D pop for fewer visible tears.
- **Spatial repair**: a same-frame-only pass (no temporal accumulation, no scene memory retained) that patches low-confidence synthesized pixels — explicitly scoped to avoid the ghosting/smearing that temporal hole-filling introduces.
- **UI-footprint guarding**: if a UI/overlay layer is composited separately (§6), the synthesis pass should never sample or modify it — gate the reprojection strength independently under the UI's known screen footprint so text/HUD elements don't get warped by scene disparity math meant for 3D geometry.

Treat this whole section as "here's the shape of the problem and the knobs a real implementation needs," not as validated numbers to copy — unlike §3/§4's compose shader (which is copy-paste-proven), single-pass synthesis quality is heavily scene- and content-dependent and needs per-title tuning no matter how carefully the base algorithm is built.

---

## 10. Pitfalls (quick reference)

| Symptom | Cause |
|---|---|
| Anaglyph black-screens the whole window when UI opens | Shared-shader overlay hits the anaglyph branch under `AlphaBlend` with hardcoded `alpha=1.0` — see §6.1. Bypass to a plain sample, or use a dedicated overlay shader (§6.2). |
| Interlace/checkerboard pattern shimmers or drifts when render-resolution changes | Pattern selected from the eye-texture's own coordinate space instead of the OUTPUT pixel coordinate, or selected before the upscale — see §3.1. |
| Interlace/checkerboard/LeiaSR soft or misaligned on a display-scaled (150%/200%) monitor even though the shader is correct | Process isn't per-monitor-DPI-aware, so the OS stretches the backbuffer after your last shader — the backbuffer isn't physical pixels. Declare DPI-awareness before the first window/swapchain — see §3.1.1. |
| HDR anaglyph looks washed out or crushed | Missing/misconfigured `paper_white` normalization around the matrix — see §5. |
| SDR title with a 10-bit swap chain renders washed out in HDR-aware code paths | Treated `R10G10B10A2` as automatically HDR; must check the swap chain's actual signaled color space, not just the pixel format — see §5. |
| LeiaSR shows bright/garbage colors | Weaver output format hardcoded instead of queried from the swap chain — see §7. |
| LeiaSR doesn't respond to head movement | `context->initialize()` called before (or without) weaver construction — see §7. |
| Exe fails to launch on machines without the SR runtime | SR DLLs hard-imported instead of `/DELAYLOAD` + `LoadLibraryW` pre-check — see `leiasr-integration`. |
| World looks squashed horizontally on a ~32:9 ultrawide (full-SbS) | Engine still rendering each eye at the full 32:9 width, then stretched into a 16:9 half. Auto-detect full-SbS and render each eye at native per-eye width + upscale (or, compositor-only, crop a centered 16:9 slice) — see §8. |
| AFR title shows a black or stale eye every other frame | Eye cache treats "not rendered this frame" as "invalid" instead of "still holding last frame's valid content" — see §2. |
| Wrong physical eye shows through the glasses/panel, and toggling "eye swap" doesn't fix interlaced/checkerboard specifically | `eye_swap` must flip inside `SampleEye` (affects every mode uniformly, including which physical buffer a given interlace phase samples) — applying it only to SbS/TaB and forgetting the line-select modes leaves those two modes unswapped. |
| Katanga/external-interop consumer never sees an image | Game still in exclusive fullscreen — these consumers typically need a windowed/bordered swap chain to composite from; see §3.2. |
| Constant-buffer values read garbage after adding a `float2` | HLSL cbuffer packing forbids vectors crossing a 16-byte boundary — pad explicitly. |
| Ghosting survives on bright HUD/subtitles over a dark scene, worst on LeiaSR | Not missing anti-crosstalk — the weaver's correction overshoots `[0,1]` and the render target clamps it. Compress contrast before the weave — see §3.4 and `leiasr-integration` A.11. |
| Ghost-reduction contrast slider washes the image out without helping | Remap applied in the wrong space (sRGB vs. linear) relative to the weaver's own pass, or the ghosting isn't clipping-driven — check the `setShaderSRGBConversion` rule in §3.4. |
| `CreateBuffer` fails, or D3D12 root constants read garbage, right after adding one shader float | The cbuffer struct is no longer a 16-byte multiple. HLSL rounds up; D3D11 requires `ByteWidth` to be a multiple of 16. Pad the tail and verify against `fxc /dumpbin` reflection, not by counting. |

---

## What this skill does NOT cover

- **The clip-space projection shear, auto-convergence control loop, and dynamic/depth-aware crosshair placement** that produce the eye textures (and, for §3.3, the `scene_shift`/`scene_scale` pair) this skill consumes — see `dynamic3d`.
- **LeiaSR weaver setup in depth** (SDK init, DLL delay-loading, HDR tone-mapping for the weaver's SDR-only input, OS window-handle plumbing) — see `leiasr-integration`; §7 here only covers where it plugs into the mode matrix.
- **Ghost reduction is owned here** (§3.4), including both levers and the refinements not to rebuild. `leiasr-integration` A.11 keeps only the SR-specific part: that the weaver cancels automatically, that its controls are unreachable on the modern interfaces, and which space its cancellation runs in.
- **HDMI 1.4 frame-packed output, dual-monitor Dual-Display output, and every anaglyph variant except Red-Cyan Dubois** (Green-Magenta Dubois, simple channel-swap, iaian7 deghosted/contrast, half-color compromise, Blue-Amber — seven in all) — deferred to `extraoutput3d`. These build directly on this skill's compose shader; the anaglyph extras are opt-in only (prefer Red-Cyan Dubois, §4), and the two output paths need extra OS-level plumbing (custom display timings / multi-monitor spanning).
- **Sky/background elements at (near-)infinite depth** — see `skybox-stereo-depth`.
- **Engine-specific render-graph wiring** (which render passes are "world" vs. "HUD," how to tag a projection for per-eye treatment, RT64's `matrixId`-based classification, etc.) — see the engine-specific sibling skill `rt64-stereo-3d` for that layer; this skill starts once you already have two eye textures per frame.
- **Head tracking.**
