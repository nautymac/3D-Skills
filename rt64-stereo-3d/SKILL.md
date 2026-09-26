---
name: rt64-stereo-3d
description: Add stereoscopic 3D (Side-by-Side, Top-and-Bottom, Row/Column Interlaced, Checkerboard, Anaglyph, LeiaSR autostereoscopic) to a project built on the RT64 renderer — i.e. an N64Recomp / RT64 game port. Covers RT64's two-queue (WorkloadQueue / PresentQueue) architecture, the two-pass per-eye render with override targets, the ProjectionProcessor stereo transforms in the clip-space separation parameterization, the trap that RT64 rescales m[0][0] for widescreen before the stereo block runs (so any shear built from it tracks the window's aspect ratio), RT64's transform-ID tagging system for classifying world vs HUD projections, the SharedQueueResources right-eye targets, the integer atomic config bridge from the UI thread and how to carry sub-unit sliders across it, forwarding ghost-reduction settings to both compose passes, and the recompilation-side patches that tag projections. Defers compose-shader internals to the output3d skill and the LeiaSR SDK side (bo3b/SR-lib api_expansion, SRInterfaceDX12, delay-load) to the leiasr-integration skill.
---

# Stereoscopic 3D for RT64-based ports

This skill captures the architecture and gotchas for adding stereo 3D to a game running on **RT64** — the renderer used by N64Recomp ports (Zelda 64: Recompiled, and similar). It assumes:

- The project is an N64Recomp port: a recompiled MIPS game running on RT64, with a `patches/` directory of `RECOMP_PATCH` C code and a host frontend (RecompFrontend / recompui) wiring RT64 to the recompiled game.
- RT64 is vendored as a library (`lib/rt64`), so you can modify its source directly.
- RT64's render path here is **D3D12 and/or Vulkan** via the Plume RHI abstraction.

For the renderer-agnostic compose-shader internals — the single fullscreen pixel shader that packs the two eye textures into SbS / TaB / interlaced / checkerboard / anaglyph output, the **anaglyph + UI-overlay alpha bug**, where the **LeiaSR weave** plugs into the mode matrix, the **per-eye 16:9 crop**, and HLSL cbuffer alignment — see the **`output3d`** skill. (Frame-packed HDMI 1.4, dual-display, and the niche anaglyph variants live in its sibling **`extraoutput3d`**.) This skill is the RT64-specific layer *around* that shader: how the two eye textures get produced in the first place, and how stereo settings flow through RT64's threading model.

If you're implementing the **LeiaSR** mode, the SDK side is a third skill: **`leiasr-integration`** covers vendoring `bo3b/SR-lib` (`api_expansion` branch — link its `SRLib::SR` CMake target and drive the weave through `SRInterfaceDX12` rather than the SDK directly), delay-load, `SRContext` lifetime, and the RT64-specific pitfalls in its **B.2.5**.

## Public reference implementation

Everything below is implemented in **[rt64-3D](https://github.com/oneup03/rt64-3D)**, a fork of RT64 with stereo output. Branches:

- **`Stereo3D`** — the generic branch this skill describes (clip-space separation, ghost reduction, LeiaSR). [BanjoRecomp3D](https://github.com/oneup03/BanjoRecomp3D) pins it.
- **`Goemon3D`**, **`DK64-3D`**, **`Dino3D`** — per-game branches used by [Goemon64Recomp3D](https://github.com/oneup03/Goemon64Recomp3D), [Donkey-Kong-64-3D](https://github.com/oneup03/Donkey-Kong-64-3D) and [dino-recomp-3D](https://github.com/oneup03/dino-recomp-3D). `DK64-3D` and `Dino3D` additionally carry the depth-driven auto-convergence loop from `dynamic3d` §4 and the per-eye aspect cap from the last section of this skill, which had not yet been merged back to `Stereo3D` at the time of writing. If a snippet here does not match `Stereo3D`, look there.

Where the pieces live (paths are the same on every branch):

| Piece | File |
|---|---|
| Mode enum, slider fields, JSON serialization | [`src/common/rt64_user_configuration.h`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/common/rt64_user_configuration.h) |
| Stereo transforms + projection classification | [`src/render/rt64_projection_processor.cpp`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/render/rt64_projection_processor.cpp) |
| Two-pass per-eye render, right-eye override targets | [`src/hle/rt64_workload_queue.cpp`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/hle/rt64_workload_queue.cpp) |
| Right-eye target storage | [`src/hle/rt64_shared_queue_resources.h`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/hle/rt64_shared_queue_resources.h) |
| Compose dispatch, UI overlay, LeiaSR intermediate + weave | [`src/hle/rt64_present_queue.cpp`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/hle/rt64_present_queue.cpp) |
| `StereoRenderer` (the `output3d` compose pass on RT64 `RenderTarget`s) | [`src/render/rt64_stereo_renderer.cpp`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/render/rt64_stereo_renderer.cpp), [`src/shaders/StereoComposePS.hlsl`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/shaders/StereoComposePS.hlsl), `src/shared/rt64_stereo_compose.h` |
| LeiaSR adapter over `SRInterfaceDX12` | [`src/render/rt64_leiasr_weaver.cpp`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/render/rt64_leiasr_weaver.cpp) |
| Host-side atomic config bridge (step 1) | Goemon64Recomp3D [`src/main/rt64_render_context.cpp`](https://github.com/oneup03/Goemon64Recomp3D/blob/dev/src/main/rt64_render_context.cpp); BanjoRecomp3D [`src/game/config.cpp`](https://github.com/oneup03/BanjoRecomp3D/blob/main/src/game/config.cpp) |
| Recompilation-side transform tagging (step 6) | Goemon64Recomp3D [`patches/transform_ids.h`](https://github.com/oneup03/Goemon64Recomp3D/blob/dev/patches/transform_ids.h) + [`patches/camera_transform_tagging.c`](https://github.com/oneup03/Goemon64Recomp3D/blob/dev/patches/camera_transform_tagging.c); BanjoRecomp3D [`patches/transform_ids.h`](https://github.com/oneup03/BanjoRecomp3D/blob/main/patches/transform_ids.h) + [`patches/projection_transform_tagging.c`](https://github.com/oneup03/BanjoRecomp3D/blob/main/patches/projection_transform_tagging.c) + [`patches/sky_transform_tagging.c`](https://github.com/oneup03/BanjoRecomp3D/blob/main/patches/sky_transform_tagging.c) |

The transform-ID constants and the `isStereo*ProjectionId` helpers are per-game by nature (the `Stereo3D` branch's are named `BANJO_PROJECTION_*`, `Goemon3D`'s `GOEMON_PROJECTION_*`); the snippets below use neutral names.

---

## RT64's two-queue architecture (the thing you must understand first)

RT64 splits frame production across two threads, each draining its own queue:

```
recompiled game ──display lists──> WorkloadQueue ──rendered targets──> PresentQueue ──> swap chain
                                   (render thread)                    (present thread)
```

- **WorkloadQueue** (`rt64_workload_queue.cpp`) — owns the *render thread*. Translates N64 display lists into draw calls, runs the **ProjectionProcessor**, and renders the scene into color RenderTargets. This is where the **per-eye render loop** lives. When framerate interpolation is on, it also generates interpolated frames here.
- **PresentQueue** (`rt64_present_queue.cpp`) — owns the *present thread*. Takes the finished color targets, runs `VIRenderer` (mono) or `StereoRenderer` (stereo) to compose them onto the swap chain, draws the config GUI overlay, and presents.
- **SharedQueueResources** (`rt64_shared_queue_resources.h`) — the struct both threads see. Holds `userConfig` (the live stereo settings), the right-eye targets, and the interpolated target arrays.

Stereo touches **both** queues: WorkloadQueue gains a second render pass per frame; PresentQueue gains the compose step. They communicate only through SharedQueueResources.

---

## Implementation order

Every step ends with something runnable. Do them in order.

### 1. UserConfiguration + the atomic config bridge

**`rt64_user_configuration.h`** — add the mode enum and the slider fields:

```cpp
enum class StereoMode {
    Off, SideBySide, TopAndBottom,
    RowInterlaced, ColumnInterlaced, Checkerboard, Anaglyph,
    LeiaSR,
    OptionCount
};
// ... in struct UserConfiguration:
StereoMode stereoMode;
uint32_t   stereoSeparation;   // 0..50 slider = 0..0.10 of screen width
uint32_t   stereoConvergence;  // TENTHS of a 0.1..50 slider, so 1..500
uint32_t   stereoHudDepth;     // 0..100 slider, 50 = screen plane
```

**Units, not just ranges.** Under the clip-space parameterization (`dynamic3d`
1.3) separation is the background disparity as a fraction of screen width, so
its slider wants a range whose top is the divergence ceiling (~0.10) rather than
an arbitrary 100. Convergence, meanwhile, is genuinely useful below 1 — but this
bridge is integer-only, and a `0.5` slider would land as `0`. Carry it in
**tenths** and convert once, on the host side of the bridge:

```cpp
// host config -> bridge
uint32_t convergence = static_cast<uint32_t>(std::lround(slider_value * 10.0));
// renderer: game units per tenth, not per slider unit
static float stereoConvergenceWorld(uint32_t tenths) { return tenths * 2.0f; }
```

Pick the per-tenth constant so whole slider values land on exactly the world
units they did before (here `slider*20` became `tenths*2`), and the change costs
existing users nothing. Document the units at every point they cross — the
config push, the bridge header, the packing comment, and the conversion — or the
next reader multiplies by 20 again.

Add a `NLOHMANN_JSON_SERIALIZE_ENUM` for `StereoMode` so it round-trips through the config file.

**The bridge** (host frontend, e.g. `rt64_render_context.cpp`) — the UI thread and the RT64 application advance on different cadences, so push config through an atomic, don't write `userConfig` directly:

```cpp
// Pack mode + 3 sliders into one atomic<uint64_t> so they land coherently.
static std::atomic<uint64_t> stereo_config_packed{0};
static std::atomic<uint64_t> stereo_config_last_applied{0};

void renderer::set_stereo_config(StereoMode mode, uint32_t sep, uint32_t conv, uint32_t hud) {
    stereo_config_packed.store(pack(mode, sep, conv, hud), std::memory_order_relaxed);
}

// Called once per frame, before the RT64 app advances (e.g. at the top of send_dl):
static void apply_pending_stereo_config(RT64::Application *app) {
    const uint64_t packed = stereo_config_packed.load(std::memory_order_relaxed);
    if (packed == stereo_config_last_applied.load(std::memory_order_relaxed)) return; // skip dup
    unpack(packed, app->userConfig.stereoMode, app->userConfig.stereoSeparation,
           app->userConfig.stereoConvergence, app->userConfig.stereoHudDepth);
    app->updateUserConfig(/*discardFBs=*/false);   // stereo doesn't change RT layout
    stereo_config_last_applied.store(packed, std::memory_order_relaxed);
}
```

`updateUserConfig(false)` propagates into `SharedQueueResources->userConfig` so both threads see it. Pass `discardFBs=false` — stereo doesn't change render-target resolution or framebuffer layout.

If you later add runtime signals from the game (see step 7), give each its own atomic and fold them into a *composite key* for the dup-skip check — a runtime flag can change frame-to-frame independently of the user's saved config:

```cpp
const uint64_t compositeKey = packed
    ^ (static_cast<uint64_t>(autoPacked) << 1)
    ^ (runtimeLowConv ? 0xA55A5AA5ull : 0ull);
```

### 2. ProjectionProcessor — the stereo math + projection classification

**`rt64_projection_processor.cpp`** is where each frame's view/projection matrices get post-processed. This is where stereo gets injected. It runs on the render thread, once per eye (the caller passes `StereoEye::Left` / `Right` / `None`).

Add three transforms (the off-axis shear, view shift, and HUD-depth math are in the `dynamic3d` skill — `applyStereoOffAxis`, `applyStereoViewShift`, `applyStereoHudShift`). The RT64-specific part is **which projections get which transform**, decided by the projection's `matrixId`:

```cpp
// World / skybox perspective projections -> off-axis + view shift (depth parallax)
if (p.stereoMode != StereoMode::Off &&
    proj.type == Projection::Type::Perspective &&
    isStereoProjectionId(curProjGroup.matrixId)) {
    applyStereoOffAxis(projMatrix, p.stereoEye, p.stereoSeparation);
    // The derived eye baseline needs tan(half horizontal FoV), which is the
    // reciprocal of the live m[0][0]. The shear above leaves [0][0] untouched,
    // so read it here, from this same transform index.
    applyStereoViewShift(viewMatrix, p.stereoEye, p.stereoSeparation,
                         p.stereoConvergence, projMatrix[0][0]);
}

// HUD: every orthographic projection, plus perspective projections explicitly
// tagged as HUD-like (pause menu, dialog overlay, title logo, ...). Untagged
// perspective projections — FMV / cutscene playback — get NEITHER, so they
// don't flicker as the per-eye shifts go opposite directions.
const bool isOrtho = (proj.type == Projection::Type::Orthographic);
const bool isHud = !isStereoProjectionId(curProjGroup.matrixId) &&
                   (isOrtho || isStereoHudProjectionId(curProjGroup.matrixId));
if (p.stereoMode != StereoMode::Off && isHud && p.stereoHudDepth != 50) {
    // Separation is a parameter here too: HUD depth scales with the depth knob,
    // which is also what makes the HUD go properly flat at separation 0.
    applyStereoHudShift(projMatrix, p.stereoEye, p.stereoHudDepth,
                        p.stereoSeparation, isOrtho);
}
```

`isStereoProjectionId` / `isStereoHudProjectionId` are small helpers you write that match the transform-ID constants the recompilation patches assign (step 6). Typically: gameplay + skybox IDs are "stereo world"; pause menu / dialog / title / game-over IDs are "stereo HUD".

**⚠ RT64 rescales `m[0][0]` before you get here, and a shear built from it
silently tracks the window's aspect ratio.** `adjustProjectionMatrix(projMatrix,
1.0f / p.aspectRatioScale)` runs a few lines *above* the stereo block — that is
RT64's widescreen hack, and in the default Expand mode `aspectRatioTarget` is
the live swap-chain ratio. So any stereo term of the form `something * m[0][0]`
(the pre-clip-space shear `(sep/2/conv) * m[0][0]`, and the perspective HUD
offset) inherits the user's window shape. Measured on a 40°-FoV N64 title, the
same sliders produced background disparity of **0.129 at 4:3, 0.097 at 16:9,
0.074 at 21:9 and 0.048 at 32:9** — half the depth on an ultrawide, with no
setting changed.

The clip-space form (`dynamic3d` 1.1) removes it from the shear by construction:
`m[2][0] += dir * separation`, no `m[0][0]` and no convergence. For anything
that legitimately still needs a projection scale — a HUD offset tuned by eye —
use a **fixed reference** value instead of the live term. For an N64 port that
constant is knowable: the game builds every perspective projection from one
`guPerspective` call at a fixed FoV, so
`1 / (target_aspect * tan(fovy/2))` at the aspect you tuned at is exact. Confirm
the game really does have one FoV before relying on it (`grep` the decomp for
the FoV setter and check every caller) — if it zooms, the reference is a lie.

RT64's own aspect handling makes this measurable rather than theoretical, which
is why it belongs here and not in `dynamic3d`.

**Critical — interpolation:** RT64 interpolates between the previous and current frame's matrices when framerate interpolation is on. The **previous-frame projection must receive the exact same stereo shift** as the current one. Find the `prevProjMatrix` / `adjustedPrevProj` path in the same function and apply the identical `applyStereoOffAxis` / `applyStereoHudShift` calls to it. Skip this and the lerp produces a half-shifted matrix that jitters every interpolated frame.

### 3. The two-pass per-eye render (WorkloadQueue)

**`rt64_workload_queue.cpp`**, in the render-thread loop, the scene currently renders once. Render it **twice** — left eye into the natural target, right eye into an *override target*:

```cpp
// LEFT eye — the existing call, now explicitly tagged.
threadRenderFrame(..., /*overrideTarget=*/nullptr, /*overrideModifier=*/0, ..., StereoEye::Left);

const auto stereoMode = ext.sharedResources->userConfig.stereoMode;
const bool stereoActive = (stereoMode != StereoMode::Off);

// RIGHT eye — re-render the same frame with the right-eye projection into a
// separate target. Gated on !usingMSAA: RT64's override-target redirection is
// itself MSAA-incompatible, so stereo + MSAA cannot coexist.
if (stereoActive && !usingMSAA && !workload.paused && !targetKey.isEmpty()) {
    RenderTarget *rightOverride = nullptr;
    uint32_t rightModifier = 0;
    if (overrideTarget != nullptr) {
        // Left used interpolatedTargets[slot]; pair with stereoRightInterpolatedTargets[slot].
        auto &slots = ext.sharedResources->stereoRightInterpolatedTargets;
        if (slots.size() <= slot) slots.resize(slot + 1);
        if (slots[slot] == nullptr)
            slots[slot] = std::make_unique<RenderTarget>(targetKey.address, Type::Color, ...);
        rightOverride = slots[slot].get();
        rightModifier = 0x8000u + (slot + 1u);
    } else {
        // Left used the natural target; right goes into the single shared target.
        auto &single = ext.sharedResources->stereoRightColorTarget;
        if (single == nullptr)
            single = std::make_unique<RenderTarget>(targetKey.address, Type::Color, ...);
        rightOverride = single.get();
        rightModifier = 0x8000u;
    }
    threadRenderFrame(..., rightOverride, rightModifier, /*uploadExtras=*/false, ..., StereoEye::Right);
}
```

`threadRenderFrame`'s `overrideTarget` parameter redirects all the scene's draws to a target you supply instead of the framebuffer-derived one; `overrideModifier` is an id salt so RT64's target cache doesn't collide the right-eye target with the left. `StereoEye` is forwarded into the ProjectionProcessor params (step 2). `uploadExtras=false` on the right pass — vertex/tile data was already uploaded by the left pass.

The `stereoActive` gate must be `stereoMode != Off`, **not** a check for a specific mode — every stereo mode (including Anaglyph and LeiaSR) needs the right-eye render. A too-specific gate is the classic "anaglyph is dim / right eye is black" bug.

### 4. SharedQueueResources — the right-eye targets

**`rt64_shared_queue_resources.h`** — add the targets the present thread will pair with the left-eye ones:

```cpp
std::unique_ptr<RenderTarget> stereoRightColorTarget;                       // pairs the natural target
std::vector<std::unique_ptr<RenderTarget>> stereoRightInterpolatedTargets;  // pairs interpolatedColorTargets
```

And reset them in the resource-reset path:

```cpp
stereoRightColorTarget.reset();
stereoRightInterpolatedTargets.clear();
```

### 5. The compose pass (PresentQueue)

**`rt64_present_queue.cpp`**, in `threadPresent`, branch on `stereoMode`:

```cpp
if (stereoMode != StereoMode::Off) {
    // Pick the right-eye target that matches the left-eye target THIS present
    // iteration is using: interpolated frame i pairs with
    // stereoRightInterpolatedTargets[i]; the natural target pairs with
    // stereoRightColorTarget. Resolve/downsample it the same way the left one
    // was, then hand both to StereoRenderer.
    StereoRenderer::RenderParams sp;
    sp.leftTexture  = /* the resolved left-eye texture */;
    sp.rightTexture = /* the resolved right-eye texture */;
    sp.stereoMode   = stereoMode;
    sp.fillFullTarget = true;     // stereo always fills the swap chain — no VI letterbox
    // cap aspectRatioTarget at 16:9 when stereo is on -- see the note below
    // forward resolutionScale-derived aspectRatioScale for the per-eye 16:9 crop
    stereoRenderer->render(sp);
} else {
    viRenderer->render(renderParams);   // unchanged mono path
}
```

`StereoRenderer` + `StereoComposePS.hlsl` + the descriptor layout + the two PSOs (Copy for the world pass, AlphaBlend for the UI overlay) + the anaglyph cross-talk matrices + the **anaglyph/UI-overlay alpha bug** + the LeiaSR weaver hand-off + the per-eye 16:9 crop — **all of that is in the `output3d` skill.** Build `StereoRenderer` per that skill; it's renderer-API-level and not RT64-specific beyond taking RT64 `RenderTarget`s as input.

**Ghost reduction lands in these same params.** `output3d` 3.4 owns the
technique (the contrast squeeze and black lift, why localizing or time-adapting
them fails, the cbuffer traps); the RT64-specific half is that the compose pass
runs **twice per frame** — once for the world, once for the UI overlay — so the
two values have to be forwarded to *both* sets of `RenderParams`. Forward them
to the world pass only and the levers appear to do nothing to subtitles and HUD,
which is exactly the content that ghosts worst. The LeiaSR path gets this free
if its compose/overlay params are copies of the base params; a separately
constructed overlay path (the non-LeiaSR branch, further down `threadPresent`)
does not. RT64 has no HDR swap-chain or screenshot path to gate the remap
against, so the SDR-only assumption `output3d` calls out holds here by default —
re-check it if either ever lands.

The one RT64-specific wrinkle: the **config GUI overlay**. RT64 draws the recompui config GUI via a `RenderHookDraw` after the world compose. In stereo it must be composed *through* `StereoRenderer` too (UI texture bound to both eye slots, `isUIOverlay=true`, AlphaBlend PSO) so it appears mirrored at zero parallax. For LeiaSR specifically the UI must be blended into the SbS intermediate *before* the weave — see the `output3d` skill's LeiaSR section.

### 6. Recompilation-side: tagging projections with transform IDs

This is the part with no analog in a normal renderer. RT64 can't tell a HUD projection from a world projection on its own — the **recompiled game tells it**, via RT64's extended GBI.

RT64 adds `gEXMatrixGroup*` display-list commands that stamp a 32-bit **transform ID** onto a group of draws. The recompilation `patches/` set a global (`cur_perspective_projection_transform_id` / `cur_ortho_projection_transform_id`) right before the game builds a projection matrix; RT64 reads that as the projection group's `matrixId`. Define an ID map (`transform_ids.h`):

```c
#define PROJECTION_GAMEPLAY_TRANSFORM_ID    0x00001000   // world  -> isStereoProjectionId
#define PROJECTION_SKYBOX_TRANSFORM_ID      0x00001001   // world  -> isStereoProjectionId
#define PROJECTION_HUD_TRANSFORM_ID         0x00001004   // HUD    -> isStereoHudProjectionId
#define PROJECTION_PAUSE_MENU_TRANSFORM_ID  0x0000100x   // HUD
#define PROJECTION_DIALOG_BUBBLE_TRANSFORM_ID 0x0000100A // HUD
// ... title logo, game-over, etc.
```

Then `RECOMP_PATCH` the game functions that set up each projection, e.g.:

```c
RECOMP_PATCH void game_draw(Gfx **gdl, Mtx **mptr, Vtx **vptr, ...) {
    cur_perspective_projection_transform_id = PROJECTION_GAMEPLAY_TRANSFORM_ID;
    // ... existing body, which builds the world perspective matrix ...
    cur_perspective_projection_transform_id = PROJECTION_HUD_TRANSFORM_ID;
    // ... existing body, which builds the HUD perspective matrix ...
    cur_perspective_projection_transform_id = 0;
}
```

`isStereoProjectionId` (RT64 side, step 2) returns true for the gameplay/skybox IDs; `isStereoHudProjectionId` returns true for the HUD/menu/dialog IDs. Untagged perspective projections (FMV/cutscene) fall through to neither and stay mono — correct, because per-eye shifts on flat full-screen video just flicker.

**Mod compatibility caveat:** modifying the *body* of an existing `RECOMP_PATCH` function shifts its compiled basic-block layout, and the recomp mod loader's runtime relink (`apply_regenlist`) can choke on that — mods that hook the same function fail to load with a generic "Code mod loading internal error". Prefer adding *new* `RECOMP_PATCH`es over editing existing ones in hot/commonly-hooked functions; if you must edit one, test mod loading after.

### 7. Runtime signals from the game (optional — scene-driven convergence)

Some scenes (menus, FMVs, first-person camera, near-camera cutscenes) read uncomfortably at the gameplay-tuned convergence. The game can classify the current scene each frame and push a bool to the host, which scales convergence down for those scenes.

> **Two generations of this.** The simple form below (a bool that multiplies convergence by a user scale) is what `Stereo3D` ships. The `DK64-3D` and `Dino3D` branches replace the multiplier with the full depth-buffer-driven loop from `dynamic3d` §4 (`stereoAutoConvergence`, `stereoConvergenceManual` as the ceiling, a signed `stereoComfortTarget` in thousandths of screen width) and keep the scene bool only as `stereoSceneLowConvergence`, which *tightens the loop's comfort budget* rather than replacing it — those scenes cut hard, and the loop's smoothing needs a few frames to settle after each cut, which is exactly when a close framing reads worst. The bridge, dup-skip and game-side detection advice is the same for both.

- A `RECOMP_PATCH` on a per-frame function (one that runs in *every* game mode — the main loop, not the gameplay-only draw) calls a bridge export each frame with a classification bool.
- The bridge writes a `std::atomic<bool>`; `apply_pending_stereo_config` reads it and, if a user-facing "auto convergence" toggle is on, multiplies `stereoConvergence` by a user scale (e.g. 0.25×) before writing `userConfig`. Under clip-space separation this gets strictly better: pulling convergence in used to *raise* background disparity (it was `sep/conv`), so the comfort fix quietly cost depth budget elsewhere. Now it only moves the screen plane.
- Fold the bool into the composite-key XOR (step 1) so the dup-skip check still catches per-scene transitions.

Detection on the game side: prefer **game-mode** checks (mode enums are state-level, mod-safe) and game-state reads (e.g. first-person camera state) over **map/level-ID** lists (a mod can repurpose a map ID for real gameplay and trip your classifier). If you must use map IDs, keep the list identical to whatever the game already uses for an equivalent purpose (e.g. its cutscene-pillarbox list) so you inherit its correctness.

**Overlay-load safety:** if the per-frame patch runs from the main loop, it may fire before lazily-loaded code overlays are mapped. Calling a function that lives in an unloaded overlay crashes on the first frames. Read the underlying BSS variable directly (`extern struct {...} D_xxxx;`) instead of calling the overlay's accessor — uninitialized BSS reads as zero, which is a safe default.

---

## RT64-specific pitfalls

| Symptom | Cause |
|---|---|
| Stereo + MSAA crashes or right eye never renders | The right-eye override-target render is gated on `!usingMSAA` because RT64's override-target redirection is MSAA-incompatible. Make MSAA and stereo mutually exclusive in the config UI; default MSAA off. |
| Anaglyph dim / one eye black, other modes fine | The two-pass gate checks a specific stereo mode instead of `stereoMode != Off`, so the right-eye pass doesn't run. |
| Interpolated frames jitter in stereo, native frames fine | The previous-frame projection (`adjustedPrevProj`) didn't get the same stereo shift as the current one; the lerp produces a half-shifted matrix. |
| Right eye shows last frame's geometry / wrong target | `stereoRightInterpolatedTargets[i]` not paired index-for-index with `interpolatedColorTargets[i]`, or `overrideModifier` collides with the left-eye target's cache key. |
| HUD has depth-parallax (should be flat) or world is flat (should have depth) | `isStereoProjectionId` / `isStereoHudProjectionId` misclassifies a `matrixId`; or the recompilation patch didn't tag that projection at all (untagged perspective = mono). |
| Config GUI shows as literal SbS / doubled through the lens | The recompui overlay was composed as a 2D pass on the swap chain instead of through `StereoRenderer` (or, for LeiaSR, after the weave instead of into the SbS intermediate). |
| Whole screen blacks out when the config GUI opens (anaglyph only) | The anaglyph/UI-overlay alpha bug — see `output3d`. The anaglyph branch must bypass to a plain `SampleEye` when `useUIOverlayMode` is set (or use a dedicated overlay shader that never runs the anaglyph branch at all). |
| Mods stop loading after a stereo change | A `RECOMP_PATCH` body edit shifted a function's layout and broke the mod loader's `apply_regenlist`. Prefer new patches over editing existing hot ones. |
| Game crashes on the first frames after boot with stereo on | A per-frame recompilation patch called into an unloaded code overlay. Read BSS directly instead of calling overlay accessors. |
| Depth effect changes with window size / ultrawide feels flat, same settings | A stereo term multiplied by the live `m[0][0]`, which RT64 has already rescaled by `1/aspectRatioScale` for widescreen. Use clip-space separation for the shear, a fixed reference scale for anything else. |
| A slider value below 1 collapses to 0 or 1 | The config bridge packs integers. Carry sub-unit sliders in tenths and convert once on the host side. |
| Ghost-reduction sliders visibly work on the world but not on HUD/subtitles | The values were forwarded to the world compose `RenderParams` but not the UI-overlay ones. Both passes need them, with identical values — the remap is affine, so matching coefficients on both layers is what makes it equal one remap of the finished composite. |
| Constant-buffer fields read garbage after adding a `float2` | HLSL cbuffer packing — see `output3d` (`_pad0` to keep `float2` off a 16-byte boundary). |
| Stereo settings don't take effect, or only after toggling something else | `apply_pending_stereo_config` not called before the frame advances, or the dup-skip composite key doesn't include a value that changed. |

---

## Cap the per-eye render aspect, or an ultrawide costs 4x

*(Implemented on the `DK64-3D` / `Dino3D` branches; not yet on `Stereo3D` at the time of writing.)*

`WorkloadQueue` derives `aspectRatioTarget` from the swap chain in the default
Expand mode:

```cpp
const float derivedRatioTarget = float(swapChainWidth) / float(swapChainHeight);
aspectRatioTarget = std::max(derivedRatioTarget, aspectRatioSource);
```

Everything else follows from that one value: `aspectRatioScale` is
`aspectRatioTarget / aspectRatioSource`, `resolutionScale.x` is
`resolutionMultiplier * aspectRatioScale`, and the render targets are sized from
that. On a 5120x1440 full-SbS panel it makes each eye render 5120x1440 -- and
then the per-eye 16:9 crop in the compose throws half of each away. Four times
the pixels of a mono 16:9 frame, to display two.

In stereo each eye is *shown* at 16:9 whatever the panel is (a full-SbS 32:9
gives each eye a native 16:9 half; a half-SbS 16:9 squeezes a 16:9 eye into
half the width and the display stretches it back), so cap it:

```cpp
if (userConfig.stereoMode != UserConfiguration::StereoMode::Off) {
    constexpr float kMaxStereoEyeAspect = 16.0f / 9.0f;
    aspectRatioTarget = std::max(aspectRatioSource,
        std::min(aspectRatioTarget, kMaxStereoEyeAspect));
}
```

Same image, half the pixels, and the compose crop becomes a no-op. Cap only the
Expand branch -- Manual is an aspect the user picked explicitly, and the crop is
what keeps that case from showing squashed halves.

Nothing reports this on its own. The crop makes the picture correct, so the only
symptom is frame time.

**Top-and-Bottom has the same bug rotated.** `resolutionMultiplier` comes from
`swapChainHeight`, and TaB gives each eye half the height, so TaB over-renders
vertically exactly as SbS was over-rendering horizontally. Fixing it halves each
eye's vertical resolution, which is a visible quality change and not only a cost
one -- decide it deliberately rather than folding it into the cap above.

---

## Cross-reference

- **`output3d`** — the compose pixel shader (8 output modes in one switch: SbS, TaB, row/column interlaced, checkerboard, external-interop passthrough, LeiaSR, and Red-Cyan Dubois anaglyph), descriptor layout, Copy vs AlphaBlend PSOs, the anaglyph cross-talk matrix, the **anaglyph + UI-overlay alpha bug** and its debugging recipe, where the **LeiaSR weave** plugs into the mode matrix, the **per-eye 16:9 crop** for ultrawide, and HLSL cbuffer alignment. Build `StereoRenderer` per that skill.
- **`extraoutput3d`** — the output paths `output3d` defers: frame-packed HDMI 1.4 (including programming the display timing), dual-monitor Dual-Display, and the seven further anaglyph variants beyond Red-Cyan Dubois. Reach for it only when one of those is explicitly asked for.
- **`leiasr-integration`** — everything SDK-side that `output3d` stops short of: vendoring `bo3b/SR-lib` (the `api_expansion` branch, whose CMake package and `SRInterfaceDX12` wrapper are what RT64 ports should link — see its **B.2.5 RT64 specifics**), the delay-loaded SR DLLs and the `LoadLibraryW`/SEH preflight, `SRContext` lifetime and init ordering, per-monitor DPI-awareness, and the command-list viewport gotcha that makes the weave render at 2× width. This is the skill for the LeiaSR mode, not `output3d`.
- **`dynamic3d`** — the off-axis shear / view-shift / HUD-depth math itself (`applyStereoOffAxis`, `applyStereoViewShift`, `applyStereoHudShift`), in the **clip-space separation** parameterization this skill's step 2 assumes: the shear *is* the knob, the eye baseline is derived per frame, and separation reads as a fraction of screen width. Its section 2 is the migration path if your port still has separation in world units and a reference-FoV slider — both of which RT64's aspect rescale makes worse than they look. Also auto-convergence and dynamic crosshair placement if you want those on RT64 too (the `DK64-3D` / `Dino3D` branches have both).
- **`skybox-stereo-depth`** — what per-eye shift sky, stars and sun sprites should get; relevant to the skybox transform ID in step 6 and to any game that draws sky elements outside the tagged world projection.
