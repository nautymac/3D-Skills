---
name: vulkan-multiview-stereo
description: Add single-pass stereoscopic 3D to a VULKAN renderer — or inject it into a Vulkan app you don't own — by turning every eligible render target into a 2-layer array, enabling VK_KHR_multiview (viewMask 0b11) on the scene passes, and rewriting the last pre-rasterization SPIR-V stage so gl_Position.x gets a per-eye offset keyed on BuiltIn ViewIndex. Load it when the task mentions Vulkan stereo / multiview / ViewIndex / gl_ViewIndex, SPIR-V patching for 3D, a Vulkan ICD or layer that makes games 3D (VKS3D, Vk3DVision-style injectors), DXVK / Zink / emulator (Xenia, Ryujinx, Dolphin) Vulkan backends that need 3D, or "one eye black / right eye stale / 2D on Vulkan" symptoms. Covers the injection architecture (ICD-wrapping-ICD vs layer, which entry points must be hooked, the loader-v6 adapter-enumeration and API-version-cap traps), resource plumbing (usage-based image upgrade to arrayLayers=2, 2D→2D_ARRAY view upgrade with cube exclusion, render-pass twins with a multiview clone chosen per framebuffer, PRESENT_SRC finalLayout rewrite, dynamic-rendering viewMask that must match the pipeline's, RP2 viewMask, VRS attachment exception, shadow/depth-only pass exclusion, transfers/barriers/compute/ray-tracing that do NOT inherit multiview), the SPIR-V patch recipe (capability + SPV_KHR_multiview for SPIR-V 1.0, section ordering, ViewIndex variable + entry-point interface + Flat in FS, injection before the final OpReturn / before each EmitVertex / per mesh vertex), the per-eye formula `x += k·(1 − C·w)` and its exact mapping onto dynamic3d's `x += sep·(w − conv)`, shader classification heuristics for UI / fullscreen / sky / geometry (matrix-provenance dataflow, dot-product counts for D3D-translated shaders, pipeline-state signatures) and why they overfit, the generic sky fix (z := w plus the constant infinity disparity, stretch-to-cover for procedural sky quads) and mono-UI-by-not-patching, the fragment-side deferred/post-process fix (sampler2D → sampler2DArray with ViewIndex as layer, fetch/gather/query-size/Dref rewrites, shared-OpTypeImage cloning), output from a 2-layer image (SbS blit, D3D11-shared-texture + D3D12-fence interop for 3D Vision / D3D-only weavers), ~35 numbered pitfalls with symptom → cause → fix mined from VKS3D's ~2,950-commit history, config knobs, a debugging workflow, and how to apply it inside an engine/emulator you control. Reference implementation: ThreeDeeJay/VKS3D (GPL-3.0 — techniques described, no code copied). Defers compose/output modes to `output3d`, the separation/convergence semantics and auto-convergence to `dynamic3d`, at-infinity math to `skybox-stereo-depth`, and the SR weaver to `leiasr-integration`.
---

# vulkan-multiview-stereo — single-pass stereo on Vulkan via multiview + SPIR-V patching

> *"This Vulkan renderer (or Vulkan game / DXVK / Zink / emulator backend) draws one eye. Make it draw both, in one pass, with correct depth, and hand the pair to an output mode."*

The core idea is small: Vulkan already has a way to render the same draw into several array layers in one pass — **`VK_KHR_multiview`** (core in 1.1). Give every scene render target two layers, set `viewMask = 0b11` on the scene passes, and add a per-eye clip-space X offset in the last vertex-processing shader stage, selected by the built-in **`ViewIndex`** (`gl_ViewIndex`). The driver broadcasts each draw to both layers; geometry is processed per view; nothing is AFR, nothing is drawn twice by the CPU.

Everything hard is in the details: *which* targets and passes get two layers, what happens to the many places where an app treats an image as having one layer, how to tell a world-space vertex shader from a HUD shader when you only have SPIR-V, and how not to produce invalid SPIR-V. This skill is distilled from reading [ThreeDeeJay/VKS3D](https://github.com/ThreeDeeJay/VKS3D) (a drop-in Vulkan ICD that does all of this to arbitrary apps) at `main` r2949 (`82c9522`, 2026-10-20) plus its `dev`/`multiview`/`spirv-val` branches, and its commit history. Where this document goes beyond what the code shows, it says **(inferred)**.

**Two contexts, one recipe:**

- **(A) You own the renderer** (an engine, an emulator's Vulkan backend). You know which passes are world, which are HUD, which images are G-buffers, where the projection lives. Use §2 and §3 as a design, implement them at the source level (your shader generator emits the offset; your render-target cache allocates two layers), and skip almost all of §5's guessing. §12 is about this case.
- **(B) You are injecting into an app you don't own** (an ICD/layer like VKS3D, or a wrapper under DXVK/Zink). You must infer everything from API calls and SPIR-V. §1 (architecture), §5 (classification) and the pitfalls are written mostly for this case.

**What this skill does NOT own:** the meaning and tuning of separation/convergence, auto-convergence and dynamic crosshair (`dynamic3d`); turning the two eyes into SbS/TaB/interlaced/anaglyph/LeiaSR (`output3d`, `extraoutput3d`); the infinity-disparity math for sky elements (`skybox-stereo-depth`); the LeiaSR SDK (`leiasr-integration`).

---

## 0. Multiview vs. the alternatives — decide first

| Approach | How | Good | Bad |
|---|---|---|---|
| **Multiview single-pass** (this skill) | 2-layer targets, `viewMask=0b11`, ViewIndex in shader | One CPU submission, eye-synced, the driver may share vertex work across views; no draw duplication or state replay | Every layer-unaware operation (copies, blits, clears, barriers, compute, RT, readback) silently handles only layer 0; requires shader rewriting; multiview forbids some features (see pitfalls); passes that must stay mono need a separate non-multiview path |
| **Two-pass per eye** (`output3d` §2) | Record the scene twice with different projection, per-eye target copies | Conceptually simple; no shader rewrite if you own the projection | 2× CPU recording; every render target needs a right-eye twin and every copy must be mirrored (the OpenGL-wrapper experience: texture views, image copies and DSA blits all have to be tracked) |
| **AFR / alternating frames** | One eye per frame | Trivial | Eye desync with motion, half rate per eye; not what modern 3D players want |

VKS3D's own release notes frame the selling point exactly as "single-pass, eye-synced (not AFR/AER)". If you own the renderer and it already has a clean "draw scene" function, two-pass is often less work; multiview wins when the CPU cost or the state-replay complexity of drawing twice is the problem, or when you can't re-record (injection).

---

## 1. Injection architecture (context B)

### 1.1 ICD-wrapping-ICD vs. layer

VKS3D registers itself as an **ICD** (a fake GPU driver) under `HKLM\SOFTWARE\Khronos\Vulkan\Drivers` (or a portable JSON next to the exe), and inside it loads the real GPU ICD (`stereo_load_real_icd` in `src/stereo.c`: `STEREO_REAL_ICD` env override → walk the Khronos driver registry JSONs skipping its own → legacy OpenGL driver key → hard-coded `nvoglv64.dll`/`amdvlk64.dll`/`igvk64.dll`). It then forwards every call, intercepting a subset.

Being an ICD (rather than a layer) costs a lot of plumbing that a layer gets for free — all of these were real bugs in the history:

- **Physical-device handles must be wrapped.** The loader calls physical-device functions with whatever handle the ICD returned from `vkEnumeratePhysicalDevices`. If you return the real driver's pointer for, e.g., `vkGetPhysicalDeviceSurfaceSupportKHR`, the loader will call it with *your* wrapper and the driver dereferences garbage. VKS3D returns its own stub for every `vkGetPhysicalDevice*` name and a generic trampoline for unknown ones (`src/icd_main.c`, `src/physdev*.c`).
- **Loader interface v6+ enumerates by DXGI adapter** via `vk_icdEnumerateAdapterPhysicalDevices`. If the wrapper ICD doesn't implement it, the loader *also* loads the real driver directly, the app sees both physdevs, picks the real one, and your `vkCreateDevice` is never called — "VKS3D does nothing". Implement it, forward to the real ICD's export, wrap the results.
- **Cap the advertised API version** to what you have stubs for. VKS3D caps `vkEnumerateInstanceVersion` at 1.3 because a loader at v7 requires every core physdev function of the claimed version to be non-NULL; claiming 1.4 without 1.4 stubs makes the app see zero devices.
- **Negotiate the loader↔ICD interface with the real ICD too.** Without calling the real `vk_icdNegotiateLoaderICDInterfaceVersion`, NVIDIA leaves its WSI table uninitialized (`VK_ERROR_INITIALIZATION_FAILED` from surface queries).
- **`vkGetDeviceProcAddr` must be intercepted** and must return your wrappers for every device command you hook; otherwise the loader builds its device dispatch from the real driver and bypasses everything (the comment in `stereo_GetDeviceProcAddr` records this as "why stereo was not working").
- **Don't validate instance handles via `loaderMagic`** — the loader overwrites that word with its dispatch pointer right after `vkCreateInstance` returns. VKS3D validates by address range of its static instance array.
- **Self-detection:** export a marker symbol and skip any ICD that exports it, or you load yourself recursively.
- **Force the minimum you need:** VKS3D bumps `pApplicationInfo->apiVersion` to ≥1.1, adds `VK_KHR_get_physical_device_properties2` at instance level, and at device level appends `VK_KHR_multiview`, `VK_KHR_maintenance2`, `VK_KHR_create_renderpass2`, `VK_NVX_multiview_per_view_attributes`, external memory/semaphore (Win32) when supported, and forces `VkPhysicalDeviceMultiviewFeatures{multiview, multiviewGeometryShader, multiviewTessellationShader}` (patching an existing struct in the app's chain or prepending one).

**If you are writing a new injector, prefer an implicit/explicit *layer*** **(inferred)** — every item above except the last exists only because VKS3D chose the ICD route. The last item (forcing the feature/extension) is the same for a layer.

### 1.2 What gets hooked, and why

From `stereo_GetDeviceProcAddr` / `get_instance_proc_addr_internal` on `main`:

| Hook | Purpose |
|---|---|
| `vkCreateDevice` | enable multiview feature + extensions; create a host-visible "stereo UBO" (unused by shaders today, see §4.4) |
| `vkCreateImage` | upgrade eligible images to `arrayLayers = 2` (§2.1) |
| `vkCreateImageView` / `vkDestroyImageView` | upgrade views of upgraded images to `2D_ARRAY`, `layerCount = 2`; track them (§2.2) |
| `vkCreateRenderPass`, `vkCreateRenderPass2(KHR)` | create the app's pass **and** a multiview twin (§2.3) |
| `vkCreateFramebuffer` / `vkDestroyFramebuffer` | decide per framebuffer whether the MV twin is usable (all attachments upgraded) (§2.4) |
| `vkCmdBeginRenderPass` / `vkCmdEndRenderPass` | substitute the MV twin at begin time; track per-command-buffer state |
| `vkCmdBeginRendering(KHR)` / `vkCmdEndRendering(KHR)` | set `VkRenderingInfo::viewMask` (§2.5) |
| `vkCreateShaderModule` / `vkDestroyShaderModule` | **cache** the original SPIR-V of VS/TES/GS/FS/mesh/raygen — patching is deferred to pipeline creation |
| `vkCreateGraphicsPipelines` | classify, patch the right stage, swap in the MV render pass / viewMask, create, record per-pipeline info (§3, §5) |
| `vkCreateShadersEXT`, `vkCmdBindShadersEXT` | same for shader objects; **binary** shader code is refused with `VK_ERROR_INCOMPATIBLE_SHADER_BINARY_EXT` so the app falls back to SPIR-V |
| `vkCreateRayTracingPipelinesKHR`, `vkCmdTraceRaysKHR` | patch raygen, launch `depth = 2` (§2.8) |
| `vkCreateSwapchainKHR`, `vkGetSwapchainImagesKHR`, `vkAcquireNextImageKHR`, `vkQueuePresentKHR`, `vkDestroySwapchainKHR` | present a **fake** 1-image, 2-layer swapchain to the app; compose to the real one at present (§8) |
| `vkCmdBind*`, `vkCmdDraw*`, `vkUpdateDescriptorSets` | bookkeeping / diagnostics on `main`; `dev` adds `vkCmdBlitImage`/`CopyImage`/`ResolveImage` wrappers that widen layer counts (§2.7) |

Deferring the patch from `vkCreateShaderModule` to `vkCreateGraphicsPipelines` is the important structural choice: **the decision to patch a vertex shader depends on the pipeline** (is its render pass multiview? is it a fullscreen quad? what is its depth/blend state?), and the same module is often used in pipelines with different answers. A patched copy is created as a temporary module per pipeline. (These temporary modules are pooled and destroyed only at device destruction because an old NVIDIA driver kept referencing module SPIR-V after pipeline creation — `stereo_icd.h`, `tmp_modules`.)

---

## 2. Resource and pass plumbing

### 2.1 Which images become two layers

VKS3D's current rule (`stereo_CreateImage` in `src/swapchain.c`): stereo enabled, multiview enabled, `VK_IMAGE_TYPE_2D`, `arrayLayers == 1`, and any of

- usage has `DEPTH_STENCIL_ATTACHMENT`, or
- usage has `COLOR_ATTACHMENT` with `mipLevels == 1` and extent > 1×1, or
- usage has `STORAGE` with `mipLevels == 1` and extent > 1×1 (needed for ray-tracing output).

The image is created with `arrayLayers = 2` and remembered in per-kind tracking arrays.

The history shows the alternative that failed: originally only images **exactly at the swapchain extent** were upgraded (so shadow maps and probes stay mono). That produced "overlapping geometry" because a scene pass at a different resolution (dynamic resolution, half-res buffers) ended up half multiview and half not; `6571ca2` switched to usage-based upgrade. The cost is that *every* attachment-capable image now has two layers, including shadow maps — which is why passes, not images, decide what is stereo (§2.3).

> In an engine you own, decide this per render target: scene colour/depth/G-buffer/post targets → 2 layers; shadow maps, probes, cubemaps, LUTs, UI atlases → 1 layer. That removes most of §2.3's heuristics.

### 2.2 Which views become arrays

`stereo_CreateImageView`: if the view's image is an upgraded image (or the fake swapchain image), change `VK_IMAGE_VIEW_TYPE_2D → 2D_ARRAY` and raise `layerCount` to 2; remember the view. **Never** touch `CUBE`/`CUBE_ARRAY` views (array layers are faces there — omni shadows broke until this exclusion). Multiview requires every framebuffer attachment to have ≥2 layers in the view, so upgrading the attachment views is not optional.

But the same image is usually also *sampled* later, and that sampled view has just become `2D_ARRAY` — while the shader that samples it still declares `sampler2D`. That mismatch is the root of §7 (**fragment-side patching**) and pitfall #14.

### 2.3 Render pass twins

`stereo_CreateRenderPass` (`src/render_pass.c`) creates two passes for each app pass:

1. **The app's pass**, with any attachment whose `finalLayout == PRESENT_SRC_KHR` rewritten to `COLOR_ATTACHMENT_OPTIMAL` — because the app now renders into the fake swapchain image, which is never presented directly (§8).
2. **A multiview twin**: same attachments (same layout rewrite), plus `VkRenderPassMultiviewCreateInfo` with `viewMask = 0b11` and correlation mask `0b11` for every subpass, zero view offsets for every dependency. The twin is stored, not returned.

The twin is **not** created for passes that look like shadow passes: a single depth/stencil attachment (`200fe77`), or exactly `R32_SFLOAT` colour + depth (a variance/linear-depth shadow pass, `129d9d1`). For `VkRenderPassCreateInfo2` the twin sets `subpass.viewMask = 0b11` and `pCorrelatedViewMasks`; the `multiview` branch (`9e77600`) only does this if some graphics subpass has a colour attachment — i.e. RP2 depth-only passes stay mono there too.

**Render pass compatibility rule (spec):** a pipeline must be created against a render pass *compatible* with the one it is used in, and the multiview view masks are part of compatibility. So the MV twin has to be substituted in **three** places consistently: framebuffer creation, `vkCmdBeginRenderPass`, and `VkGraphicsPipelineCreateInfo::renderPass`. Miss one and you get validation errors, a device loss, or a pipeline that renders only view 0.

(inferred) The RP1 twin is built by replacing the create-info's `pNext` with the multiview struct; any app structs in that chain (input-attachment aspect info, fragment density map, …) are dropped. If you write this yourself, *insert* into the chain instead.

### 2.4 Per-framebuffer decision

`stereo_CreateFramebuffer` (`src/framebuffer.c`): use the MV twin only if **every** attachment view is an upgraded view — the single allowed exception is the render pass's **fragment-shading-rate attachment**, which is not a per-eye image (VRS sample fixed in `3d30770`). "Any attachment upgraded" must never be enough. The framebuffer remembers its original pass and the twin.

At `vkCmdBeginRenderPass`, a framebuffer tracked as MV with a matching original pass gets its twin; a framebuffer tracked as **non-MV is authoritative** and is never promoted just because the requested pass has a twin (that bug promoted shadow framebuffers). An app may legally begin a framebuffer with a different-but-compatible pass; VKS3D then looks up that pass's twin.

### 2.5 Dynamic rendering (`VK_KHR_dynamic_rendering` / 1.3)

There is no render pass object — the view mask lives in two places that **must agree**:

- `VkRenderingInfo::viewMask` at `vkCmdBeginRendering`, and
- `VkPipelineRenderingCreateInfo::viewMask` in the pipeline's `pNext`.

`main` sets both to `0b11` whenever the app left them at 0 (`d57f3b5`, `7693291`). That is too blunt: every dynamic-rendering pass becomes multiview, including shadow passes whose single-layer attachments… are actually two-layer now (§2.1), which is perhaps why it mostly works (inferred). The `multiview` branch removes both forcings (`4efd827`, `914177a`) pending a per-pass decision. Two more traps live here:

- `main` writes the new `viewMask` **into the app's const `VkPipelineRenderingCreateInfo`** in place. Don't mutate app memory; copy the chain.
- Pipeline-side and begin-side decisions are made at different times (pipeline creation vs. recording) and by different code. If you make the decision per pass, record it per pipeline so a pipeline created for a mono pass isn't used inside a multiview one (that is an error by the compatibility rule).

**Graphics pipeline libraries** (`VK_EXT_graphics_pipeline_library`, used heavily by DXVK 2.x): the pre-rasterization library carries the vertex shader, the fragment-output library carries the formats; the view mask must be the same in every library and in the linked pipeline (spec rule, stated here from the spec, not from VKS3D code). VKS3D handles the fragment-only library case by routing it to the FS path (`stereo_CreateGraphicsPipelines`, "GPL fragment shader bit"); the Vulkan Samples GPL demo was "fixed in r2630".

### 2.6 MSAA, VRS, subpasses

- MSAA colour/depth images upgrade like any other (2-layer multisampled arrays are fine). Fragment-side sampling of `sampler2DMS` → `sampler2DMSArray` must change the **Arrayed** operand of `OpTypeImage`, which is word 5; an off-by-one that checked word 6 (MS) broke every MSAA case (`58ce78e`).
- VRS: see §2.4 (shading-rate attachment exception).
- Subpasses and input attachments: inside a multiview subpass, input attachment reads are implicitly from the current view's layer (spec behaviour), so a subpass-based deferred pipeline needs no fragment patching for its input attachments — only for *sampled* G-buffer reads across passes (§7).

### 2.7 Everything that is NOT multiview

Multiview only affects draws inside a multiview subpass/rendering scope. Every other operation that touches an upgraded image uses whatever layer range the app passed — almost always `baseArrayLayer = 0, layerCount = 1`. The right eye then shows stale data (often black, or last frame), or the image ends up half in the wrong layout.

| Operation | What breaks | Fix |
|---|---|---|
| `vkCmdBlitImage`, `vkCmdCopyImage`, `vkCmdResolveImage` between two upgraded images | only layer 0 copied → right eye stale/black after post-processing chains, downsample, resolve | widen `layerCount` 1→2 on both subresources when both images are upgraded — implemented on `dev` (`3fbd243`), not on `main` |
| Copy/blit between an upgraded and a mono image | ambiguous | decide per case: mono→stereo should fill both layers (two regions); stereo→mono (screenshot/readback) should take layer 0 or produce SbS |
| `vkCmdPipelineBarrier` / image barriers with `layerCount = 1` | layer 1 never transitioned (inferred; not hooked on `main`) | widen to `VK_REMAINING_ARRAY_LAYERS` for upgraded images |
| `vkCmdClearColorImage` / `ClearDepthStencilImage` | layer 1 not cleared (inferred) | widen range |
| `vkCmdCopyImageToBuffer` (screenshots, readback) | "only dumps one eye", "screenshot may crash" in VKS3D's compat list | decide policy; buffer must be sized for the layers you copy |
| **Compute** writing a storage image | not multiview at all; only layer 0 written | dispatch per layer (z = 2 and use `GlobalInvocationID.z` as eye, or two dispatches with a push constant) and patch the image type to arrayed; VKS3D does not patch compute (GPU-particle and OIT samples stay mono) |
| **Ray tracing** | not multiview | VKS3D launches `depth = 2` when the app used 1, patches raygen to use `LaunchID.z` as eye and to write layer `z` (§2.8) |
| Host image copy, sparse binding, external import | layer count assumptions | out of scope for VKS3D |

### 2.8 Ray tracing (raygen)

`spirv_patch_stereo_raygen` + `stereo_CmdTraceRaysKHR`: the launch is widened to `depth = 2`; in raygen, `LaunchID.z` selects the eye; the eye's horizontal displacement is applied to the ray origin in camera space (`−offset / P[0][0]`, transformed by the inverse view matrix) and the off-axis frustum shift to the ray's NDC x (`+offset·C`); `OpImageRead/Write` coordinates gain `z`. After the trace, VKS3D copies both layers of the most recently created storage image into the fake swapchain image — a fragile guess at "which image is the RT output". Eye order and sign were each wrong once (`391c452`, `065e33c`). Treat this as a proof of concept; in an engine you own, do it in your raygen source.

---

## 3. The SPIR-V patch — recipe

### 3.1 Which stage

Patch the **last pre-rasterization stage** of each graphics pipeline: mesh shader if present, else geometry, else tessellation evaluation, else vertex. (VKS3D's order in `stereo_CreateGraphicsPipelines`: fullscreen-FS route → mesh → GS → TES "Path A" → VS "Path B". An older approach injected a whole TCS+TES pair to carry the offset — `src/tes_inject.c` — as a workaround for one old driver; it caused interface-mismatch crashes on newer drivers and is no longer used.)

Patch only if the pipeline will run in a multiview pass (MV twin render pass, or non-zero pipeline `viewMask`). A shader patched to read `ViewIndex` in a non-multiview pass just gets view 0 — harmless but it costs a module.

### 3.2 Module surgery — the validity checklist

Binary SPIR-V has strict section order: capabilities → extensions → ext-inst imports → memory model → entry points → execution modes → debug → **annotations** → types/constants/globals → functions. Every "Fix … to pass SPIR-V validation" commit in the history is one of these:

1. **Capability.** Add `OpCapability MultiView` if absent, among the capabilities (`aa42525`).
2. **Extension.** For SPIR-V **1.0–1.2** modules, also add `OpExtension "SPV_KHR_multiview"`, *after the last capability* (`d6632fc`, `1d071be`). (From 1.3 multiview is core SPIR-V.)
3. **ViewIndex variable.** If the module doesn't already have one (reuse it if it does — `476e3b7`), add `%int` (signed 32-bit) if missing, `%ptr_Input_int`, `OpVariable %ptr_Input_int Input`, and `OpDecorate %vi BuiltIn ViewIndex` **in the annotation section** (first OpDecorate position is safe; emitting it next to OpEntryPoint is not — `9ac851a`).
4. **Fragment shaders: decorate `Flat`.** An integer Input in a fragment shader must be `Flat` (`fcc8e94`).
5. **Entry-point interface.** Append the new variable's ID to the `OpEntryPoint` of *your* entry point (for SPIR-V ≥ 1.4 all referenced globals must be listed; before 1.4, Input/Output must be). A module can have several entry points; extend only the one you patch.
6. **New types/constants** go after the existing type section, before the first `OpFunction`; never duplicate a non-aggregate type that already exists (`OpTypeBool`, `OpTypeVector %float 3`, `OpTypeInt 32 1` — "duplicate non-aggregate type" was fixed repeatedly: `265b19a`, `ef6c688`, `66055b7`, `53d5310`). Look up existing IDs first; allocate new IDs from the header's `bound` and **write the new bound back** (word 3).
7. **Don't clone result IDs.** A rewritten instruction that keeps its result ID must be emitted once; "multiple definition" errors and an infinite-loop hang (`96baf05` — a loop that `continue`d without advancing the word cursor) came from rewrite paths that forgot to skip the original.
8. **Normalize vector `OpSelect` with a scalar condition.** A scalar bool condition for a vector result is only legal from SPIR-V 1.4; VKS3D splats such conditions into a bool vector for every `vec4` select it re-emits (`e816e3e`).
9. **Re-check store types after you retype things.** VKS3D repairs any `OpStore` of a `vec4` into a `vec3` pointer that its own rewrites produced (`a18bc17`, `cd4b394`). Better: don't produce them.
10. **Validate every patched module offline** (`spirv-val`) during development — VKS3D's `scripts/debug.cmd` dumps original (`-vs.spv`) and patched (`+vs.spv`) modules and runs `spirv-dis`/`spirv-val` on them; a full `spirv-val` branch exists.

### 3.3 Where the offset goes

- **VS / TES:** in the function that stores `Position` (`position_function` — not always the entry point), immediately **before the final `OpReturn`**: load `Position` (through the `gl_PerVertex` block member access chain if Position is a block member — scan twice, because some modules declare the pointer type before the `OpMemberDecorate BuiltIn Position`), compute the new x, store back.
  - (inferred) Only the last `OpReturn` is instrumented, so a shader with an early return from the position-writing function would skip the offset on that path. A structurally safer variant is to wrap: rename the entry point's body into a function and call it from a new entry that applies the offset after the call.
- **GS:** before **every** `OpEmitVertex` (each emitted vertex has its own Position).
- **Mesh:** per written vertex position in `gl_MeshVerticesEXT[].gl_Position`.

Pseudo-code (own wording; illustrative):

```c
// emitted right before the final OpReturn / each OpEmitVertex
vec4  p   = load(Position);
float k   = (load(ViewIndex) == 0) ? K_VIEW0 : K_VIEW1;  // constants, see §4
p.x       = p.x + k * (1.0 - C * p.w);                    // VKS3D's off-axis form
store(Position, p);
```

`K_VIEW0`, `K_VIEW1` and `C` are **`OpConstant`s baked into the patched module** at pipeline-creation time (§4.4 for why that matters).

### 3.4 What VKS3D does *not* do (and you might)

- It doesn't touch the projection matrix or the uniform data. Clip-space post-offset only. (An abandoned direction tried to find the camera UBO in SPIR-V and rewrite the bound descriptor — the remnants are `proj_set/proj_binding/proj_member_mask` and the always-false `rewrite_proj` in `stereo_CmdBindDescriptorSets`.)
- It doesn't use `VK_NVX_multiview_per_view_attributes` / per-view position outputs, even though it enables the extension (inferred from the absence of any use in `shader.c`).
- No specialization constants or push constants for the eye parameters.

---

## 4. Stereo math and parameter mapping

### 4.1 The formula actually emitted

For the normal (non-sky) path, `emit_body` in `src/shader.c` produces, per vertex:

```
x' = x + k · (1 − C · w)          k = per-view constant, C = "convergence"
```

with `k_view0 = +separation/2`, `k_view1 = −separation/2` in the default off-axis mode (no flip). NDC shift per eye: `k/w − k·C`. So:

- **Zero parallax at `w = 1/C`** (in the app's clip-w units, i.e. view-space depth in *app world units* — per game).
- **Far field** (`w → ∞`): per-eye NDC shift → `−k·C`. View 0 shifts **left** by `separation·C/2`, view 1 right — uncrossed disparity, so view 0 is the left eye and layer 0 goes to the left half.
- **Near field** (`w → 0`): `k/w` grows without bound (crossed disparity, as expected in front of the screen plane).

**The README is stale** here: it documents `x += offset · w` with `offset = ±(sep − conv)/2`, and `test_stereo_math.c` tests only that older offset arithmetic (and still asserts a 16-byte `StereoUBO`, which is now 32). Read `emit_body`, not the docs.

### 4.2 Mapping onto `dynamic3d`'s convention

`dynamic3d` writes stereo as `x += sep_eye · (w − conv)`, with `sep_eye` signed per eye (left negative, 3Dmigoto convention) and `|sep|` = the per-eye NDC shift at infinity = fraction of screen width between the two eyes' images of an infinitely distant point. Rewrite VKS3D's form:

```
x + k(1 − C·w)  =  x − k·C · (w − 1/C)
⇒  sep_eye  = −k·C      → left (view 0): −separation·C/2,  right: +separation·C/2
   conv     = 1/C
   background disparity (fraction of screen width) = separation · C / 2
```

So VKS3D's two knobs are not dynamic3d's two knobs:

- VKS3D `convergence` changes **both** the screen-plane depth (`1/C`) **and** the background disparity (`separation·C/2`). Raising convergence to pull the screen plane closer also widens the infinity disparity — the coupling `dynamic3d` §1.3/§4.3 exists to avoid. If you expose controls in your own implementation, expose `sep_eye` and `conv` in the dynamic3d form and *derive* `k` and `C` (`C = 1/conv`, `k = ∓sep/C` per eye) if you want to keep VKS3D's emitted shape. Or just emit `x += sep_eye*(w − conv)` directly — it's the same instruction count.
- The code defaults (`separation = 0.065`, `convergence = 0.030`) give background disparity ≈ 0.001 of screen width — nearly flat; the shipped `vks3d.ini` uses `separation = 0.5`, `convergence = 0.1` (background 2.5% of width, screen plane at w = 10), after `ebcac92` "Increase default separation and convergence". Sanity checks that clamped convergence < separation were commented out (`2379e10`) because under this formula they're meaningless.
- `dynamic3d`'s divergence bound (`sep ≲ IPD / screen_width`, ~0.1 on a 27″) translates to `separation·C/2 ≲ 0.1`.

### 4.3 Modes and flags

- `projection = off-axis | parallel` in the ini. In "parallel", `k = ±(separation − convergence)/2` — but `emit_body` no longer branches on the mode, so the `(1 − C·w)` term is still emitted; and the key is read through the **boolean** INI reader (`cfg_bool("projection", 1)` in `stereo_config_init`), which returns the default for any string other than 1/0/true/false/yes/no — so `projection=parallel` is silently ignored and the mode is always off-axis (inferred from the code; a stringly-typed option parsed as bool).
- `flip_eyes` swaps `k_view0/k_view1`, which negates the whole offset — a correct eye swap for this formula.
- **Eye order was flip-flopped at least four times** (`01ef4e2`, `e9710d2`, `acf50eb`, `2dd9f91`, plus RT `391c452`): the selected constant for view 0 is the *positive* one, which reads "wrong" if you name constants by their near-field sign. Name per-view constants by what they do at infinity, and verify empirically (dynamic3d §1.4's shear sign check: an object nearer than the screen plane must show crossed disparity).

### 4.4 Baked constants vs. live parameters

The eye constants are `OpConstant`s in each patched module. Consequences:

- **Changing separation/convergence at runtime does nothing to existing pipelines.** VKS3D's Ctrl+F3–F6 hotkeys update `StereoConfig` and write the stereo UBO — but no patched shader reads that UBO, so only pipelines created afterwards change. The release notes list "Hotkeys don't work" as a known issue; this is the reason.
- Pipeline caches key on SPIR-V, so different settings produce different cache entries (fine), and apps that cache pipelines by their *own* key (DXVK state cache, emulator caches) keep using whatever was compiled first (inferred).

For live parameters an injector needs a channel the app doesn't know about: an extra descriptor set index above the app's highest (requires rewriting pipeline layouts), an appended push-constant range (requires rewriting layouts and every `vkCmdPushConstants` stays valid), or a buffer-device-address pointer baked as a constant into the module and read with `PhysicalStorageBuffer` loads (no layout change; needs `bufferDeviceAddress`) **(inferred design options — VKS3D implements none)**. In an engine you own, put the two floats in an existing per-frame UBO or push constant.

---

## 5. Shader classification (context B)

The question per pipeline is: **world geometry (stereo), screen-space UI/HUD (leave mono → screen depth), fullscreen post/deferred pass (don't move vertices; fix the fragment side, §7), or sky (special depth, §6)?** With only SPIR-V and pipeline state, VKS3D answers it with dataflow signals and state signatures. They are worth knowing because the same signals help when a renderer you own lacks semantic tags — but read §5.3 before copying any of them.

### 5.1 Signals computed per shader (`do_scan` / `spv_scan`)

- **Matrix provenance.** A forward dataflow over SSA IDs marks values derived from matrix-typed loads (struct members of `OpTypeMatrix`, pointers to them) through `Load`, `AccessChain`, `CompositeExtract/Construct/Insert`, `VectorShuffle`, `Transpose`, `F{Add,Sub,Mul,Div}`, `Select`, `CopyObject`, `Bitcast`, `ExtInst`; and records `OpMatrixTimesVector/Matrix`, `VectorTimesMatrix`. A store to `Position` whose value is *not* matrix-derived sets `has_direct_position_write`. Deliberately **not** propagated: "projection" provenance through `MatrixTimesVector`, because fragment shaders (SSAO, SSR, depth reconstruction) multiply arbitrary vectors by the projection without producing clip positions.
- **`dot_count`** — number of `OpDot`. D3D bytecode translated to SPIR-V (DXVK; also any emulator translating guest shaders) implements matrix×vector as four `dp4` → four `OpDot` on vec4 constants: **no matrix types at all**. `5130c69` ("Fix black screen when using DXVK") is the moment this was learned: the "skip shaders without matrix ops" rule mono-ized every DXVK vertex shader. UI rules later used `dot_count ≤ 2` instead (`de6b22b`).
- **Position input shape** — `has_v2_position_input` (a `vec2` Input: typical of 2D UI vertex formats); Zink/OpenGL UI uses `vec4` position inputs, hence a separate `has_position_input` (`0fb2717`).
- **Location-0 screen values** — values loaded from the `Location 0` input; `screen_has_zw_use` when a shuffle of them reads `.z`/`.w` (`0af8999`): 2D UI passes x,y through, 3D content uses z/w.
- **Position block vs. bare variable**, `EmitVertex` count, GS input/output modes and vertex counts, a "projection UBO" guess (a struct with matrix members in a Uniform variable, plus which member indices are used — note the hard-coded member meaning `0 view, 1 viewInv, 2 proj, 3 projInv, 4 viewProj, 5 prevViewProj` matches one sample framework's UBO layout, not a general truth).
- **Constant-pattern probes** — e.g. a Position built as `vec4(xy, 1.0, 1.0)` ("z = 1 position": a background quad at the far plane).

### 5.2 Rules (current `main`)

Applied in `stereo_CreateGraphicsPipelines` (pipeline-level) and `spirv_patch_stereo_vertex` (shader-level), only when `mono_ui = true`:

- **Not a multiview pass** → don't patch.
- **UI candidate (shader-level)** — VS only, no `EmitVertex`, and one of: block Position + no matrix/vector-matrix ops + vec2 position input + `dot_count ≤ 2` + no z/w use; or Location-0 screen values + direct Position write + no z/w use; or two narrower "projection UBO with exactly 2 dots" variants. → leave mono.
- **UI candidate (pipeline-level)** — combinations of: depth test **and** write disabled + screen-space VS; or "passthrough" VS + 1 binding/3 attributes + triangle list + classic alpha blend (`SRC_ALPHA, ONE_MINUS_SRC_ALPHA`); or depth test on / write off / compare `ALWAYS` + cull none + an exact attribute layout (pos `R32G32B32_SFLOAT`@0, colour `R8G8B8A8_UNORM`@12, uv @16); or a geometry shader with 7 input locations emitting a 4-vertex strip with depth test off (a sprite/text GS). → leave mono.
- **Fullscreen/post pass** — no vertex bindings (`is_quad`) and a VS with neither matrix ops nor direct position writes, or a vec2-position fullscreen VS that is not screen-space UI, or a GPL fragment-only library — and no GS/TES/TCS/mesh → **do not patch the VS; patch the FS** (§7).
- **`vs_uv_quad`** (a quad VS forwarding exactly two location-0 values, no z=1) → skip. **`vs_pure_quad`** (quad VS without user outputs) → skip.
- **Background/sky** — see §6.
- Everything else in a multiview pass → patch as world geometry.

### 5.3 What the history says about these heuristics

- **They overfit.** The last month of `main` (`2178e6a`…`82c9522`) is ~100 commits of narrowing and broadening UI gates per title ("Tighten intro logo mono UI routing", "Detect mono UI menu background", exact attribute-format matches, "UI skip dot_count<8" → "<18" → reverted). Every rule that fixed one game broke another — "Fix sky false positive flattening stereo", "Preserve world-transformed UI geometry", "Avoid broad shared-shader UI skip".
- **Shared shaders defeat shader-level rules.** Many engines draw HUD *and* world with one shader (a world-space HUD, a 3D minimap). The only fully reliable discriminator is per-draw (which matrix is bound, which pass) — which an injector at the pipeline level cannot see.
- **The escape hatches are mandatory**: `mono_ui=false` (never classify), a per-shader-hash skip list (`VKS3D_SKIP_SHADER_PATCHES=hash,hash` or `*`; hash = FNV-1a over the module words, same as the dump filenames), and SPIR-V dumping to find the hash. Expose the same three in anything you build.
- **If you own the renderer, classify semantically** — the projection type (perspective vs. ortho), the pass, the render target, explicit HUD tags (cf. `rt64-stereo-3d`'s transform-ID tagging). Use §5.1's signals only as a fallback for content you can't tag (e.g. guest shaders in an emulator, where the D3D-style `dot_count`/z-w-use signals are the relevant ones).

---

## 6. Generic depth fixes: mono UI and sky

### 6.1 UI at screen depth = "don't patch"

A vertex shader left unpatched in a multiview pass produces identical positions in both views → zero disparity → the UI sits exactly on the screen plane. No extra math. The failure modes are classification errors (§5.3): world geometry flattened (whole scene at screen depth — the troubleshooting tree literally says "everything is at screen depth → set `mono_ui=false`"), or UI patched (HUD floats at whatever depth its w implies — usually w = 1 → strongly crossed under `(1 − C·w)` with small C). For depth-placed HUD (crosshair at aim depth), see `dynamic3d` §5.

### 6.2 Sky / background (`sky_flatten`, `sky_extend`)

Detected as "background" (`vs_background` in `stereo_CreateGraphicsPipelines`) when depth **writes are off** and either the VS is a quad with a vec3 user output (a direction for a cubemap lookup) or Position is built with z = 1, or the VS has matrix ops, triangle list, cull none with depth test off / or front-face culling (an inside-out skybox). Then the patch replaces the normal offset:

- **Matrix skybox (flatten):** `x' = x − k·C·w` and **`z' = w`**. In NDC that is exactly the far-field limit of §4.1 (`−k·C`), applied regardless of the skybox's actual w — a skybox drawn at a small radius around the camera no longer pops out — and depth 1.0 so it never occludes. This is `skybox-stereo-depth` §6 strategy B (analytic infinity constant), injected generically. (`2ab68d1` made the disparity constant, `2e3a559` fixed its sign, `f04be32` forces max depth.)
- **Procedural / flat quad sky (extend):** `x' = x·(1 + E) − k·C` with `E = |k·C|` = separation·C/2, plus `z' = w`. A fullscreen quad shifted by the infinity disparity would expose a black strip at one edge; scaling by `1 + E` keeps `[−1, 1]` covered after the shift (`17f61cc`, `2f0afd7` restrict the stretch to direct-position shaders).

Caveats (inferred): `z' = w` means "far" only for **standard** depth (`[0,1]`, less-compare). With **reversed-Z** (far = 0, `GREATER` compare — common in modern engines and in DXVK titles that use it) it puts the sky at the *near* plane. Check the pass's depth compare op and use `z' = 0` there. Also note the procedural stretch changes the sky's apparent FoV slightly (it is a zoom by `1 + E`).

---

## 7. Fragment side: deferred and post-process passes

When the G-buffer is a 2-layer array, a fullscreen lighting/post pass running in a multiview pass must read **its own eye's layer**. VKS3D's FS patcher (`spirv_patch_stereo_fs`) does, for each image variable it judges to be a stereo attachment:

- `OpTypeImage … Arrayed = 0` → `1` — but **clone** the type rather than editing it in place when it is shared: SPIR-V deduplicates types, so one `OpTypeImage %float 2D 0 0 0 1 Unknown` is used by the G-buffer *and* by every material texture; editing it in place arrays them all. The pointer type, `OpTypeSampledImage` and every `OpLoad`/`OpSampledImage`/`OpImage` result type along the chain must follow (the `spirv-val` work from `e83aec5` to `0164c1d` is ~150 commits of exactly this, including descriptor arrays and images passed through function parameters).
- Sample coordinates `vec2 uv` → `vec3(uv, float(ViewIndex))`; `OpImageFetch` `ivec2` → `ivec3(xy, ViewIndex)`; `OpImageGather` likewise; `OpImageQuerySize(Lod)` now returns `ivec3` → shuffle back to `ivec2` for the original consumer.
- **Dref (shadow) samplers**: an arrayed depth-compare sample uses `vec4(uv, layer, dref)`; and a shadow map is normally *not* stereo — making it arrayed was tried and reverted (`d8f25db`/`3bc12b1`).
- **Projective sampling** (`OpImageSample*Proj*`) is illegal on arrayed images in SPIR-V; it must be rewritten to a divide + non-projective sample (attempted in `145c249`, reverted).

**The hard problem — which descriptors are stereo?** The FS is patched at pipeline creation; the images are bound later via descriptors. VKS3D currently answers with a **binding-number heuristic**: bindings 0–4 are treated as G-buffer attachments (position/depth, normal, albedo, specular, SSAO — one sample framework's deferred layout), everything else stays `sampler2D`. The `dev` branch tried and reverted turning that off several times (`0290ba4`, `d956e80`, `a25d205`). A robust design (inferred): track upgraded views through `vkUpdateDescriptorSets`/templates/push descriptors/descriptor buffers keyed by `(set layout, binding)`, and patch at pipeline creation using the pipeline layout's set layouts; or, in an engine you own, simply generate the arrayed variant for passes that read eye targets.

**Mismatch consequences:** a 2D_ARRAY view bound to a `sampler2D` declaration (e.g. a material that samples a render-to-texture mirror, or a shadow map whose image got upgraded) is a descriptor/view-type mismatch — undefined behaviour that some drivers tolerate by reading layer 0 (inferred; VKS3D's compatibility list shows "Offscreen rendering: reflection is flat", "SSAO offset with distance", "HDRCubeMap reflection doubled" — the symptom family).

Screen-space effects that reconstruct position from depth with the *mono* projection (SSAO, SSR, radial blur's centre, depth-of-field focus, parallax occlusion) will be slightly wrong per eye even when they sample the right layer: their inverse projection doesn't know about the per-eye offset. VKS3D's compatibility notes record exactly those ("AO offset with distance to convergence", "radial blur end point shifted with distance", "no POM stereo displacement").

---

## 8. Output — from a 2-layer image to the display

### 8.1 The fake swapchain

In SbS/TaB/interlaced modes `stereo_CreateSwapchainKHR` creates the **real** swapchain itself (`gpu_compose_sc_init` in `src/present_alt.c`: surface caps' current extent, `TRANSFER_DST` usage, MAILBOX if available else FIFO, colour space `SRGB_NONLINEAR`) and gives the app a **single** 2-layer image (`GetSwapchainImages` returns 1 image, `AcquireNextImage` always returns index 0). Acquire waits on the previous compose's fence (a CPU stall the README acknowledges) and signals the app's semaphore/fence with an empty submit. Present records: barriers → two `vkCmdBlitImage` (layer 0 → left half, layer 1 → right half, linear filter, i.e. **half-width SbS**) → present the real image. `shaders/composite.*.glsl` (a `sampler2DArray` SbS shader) is compiled but the blit path is what runs. `tab` and `interlaced` are marked BROKEN in `vks3d.ini`; DXGI 1.2 stereo and D3D9Ex/NvAPI modes too.

Traps (inferred unless noted): the blit's destination uses the *app's* extent while the real swapchain uses the surface's current extent — they diverge after a resize; HDR swapchain formats/colour spaces are dropped (`SRGB_NONLINEAR` hard-coded); one fake image means no CPU/GPU overlap. Resizes reuse the `StereoSwapchain` slot through `oldSwapchain` (`f5123da`) and must not destroy it in the subsequent `vkDestroySwapchainKHR` of the old handle (`resize_reused`) — two separate bugs ("crash on resize", "black view after resize", r500/r538).

**Better in your own implementation:** don't blit. Run `output3d`'s compose shader with the eye pair bound as **one `sampler2DArray`** (`SampleEye(layer)`), which gives every mode (full SbS, TaB, interlaced at display-native resolution, anaglyph, LeiaSR input) from the same 2-layer image without copies. Keep N real swapchain images and per-frame fences.

### 8.2 3D Vision (NV3D-Lib) — the D3D interop pattern

`presentation_mode = nv3dlib` (`src/present_nv3d.cpp`, submodule `oneup03/NV3D-Lib`): NV3D-Lib creates a **D3D11 shared texture (2W×H, `B8G8R8A8_UNORM`) and a D3D12-style fence**, returning NT handles. VKS3D imports the texture with `VK_KHR_external_memory_win32` (`D3D11_TEXTURE_BIT`) into a `VkImage`, imports the fence as a **timeline semaphore** (`D3D12_FENCE_BIT`), and per frame: copies layer 0 → left half, layer 1 → right half, signals timeline value *n*, and calls `Present(n)`; the D3D side waits for *n* and presents through 3D Vision. The app's acquire is satisfied with an empty submit.

**This is the general recipe for any D3D-only consumer** — e.g. a D3D11/D3D12 LeiaSR weaver when the producer is Vulkan (see `leiasr-integration` B.4): create the full-SbS texture on the D3D side, import into Vulkan, write it, signal a shared fence, weave/present on the D3D side.

---

## 9. Pitfalls — symptom → cause → fix

Numbers in parentheses are VKS3D commits (`main` unless noted). "(i)" = inferred from reading, not demonstrated by a commit.

1. **Injector loads but nothing changes.** Loader v6 DXGI-adapter enumeration bypasses a wrapper ICD that doesn't implement `vk_icdEnumerateAdapterPhysicalDevices`; or `vkGetDeviceProcAddr` isn't intercepted. → §1.1.
2. **App sees no GPU after a driver update.** Wrapper advertises an API version it lacks stubs for. → cap the version.
3. **Whole scene 2D under DXVK / translated shaders.** "Has matrix ops" used as the geometry test; D3D-translated VS have only `OpDot`s. (5130c69) → never require matrix types; use dot-product / z-w-use signals.
4. **Everything at screen depth.** UI classifier false positive on world shaders. → `mono_ui=false` escape hatch, per-hash skip list.
5. **HUD floats in front of the screen.** UI shader patched. With `(1 − C·w)` and HUD w = 1 the HUD gets a large crossed disparity. → classify, or tag HUD in your renderer.
6. **Overlapping/duplicated geometry or shadows.** Half the targets of a pass upgraded (extent-based upgrade), or fullscreen/composite VS patched. (6571ca2, 2de7e30) → usage-based upgrade; never patch fullscreen VS; fix the FS instead.
7. **Duplicated / offset shadows.** Shadow pass made multiview, or the stereo pipeline used in it. (200fe77, 129d9d1, d44efbd: `in_mv_rp` must not default to "multiview enabled") → exclude depth-only and shadow-signature passes; pipelines inherit the pass decision.
8. **`VK_ERROR_DEVICE_LOST` on the first multiview pass.** An attachment with one layer in a `viewMask=0b11` pass (the original reason `multiview` was default-off; see the ini warning). → MV twin only if *all* attachment views are 2-layer (VRS attachment excepted).
9. **Shadow framebuffer promoted to multiview.** Begin-time lookup by render pass alone. → a framebuffer tracked as non-MV is authoritative.
10. **Validation: render pass incompatible with pipeline.** MV twin substituted at begin but not at pipeline creation (or vice versa). → substitute in framebuffer, begin, and pipeline, consistently.
11. **Black screen after present.** App's render pass still ends in `PRESENT_SRC_KHR` on the fake swapchain image. → rewrite `finalLayout` in every pass (both twins).
12. **Dynamic-rendering title renders mono or black.** `VkRenderingInfo::viewMask` and `VkPipelineRenderingCreateInfo::viewMask` disagree, or neither is set. (d57f3b5, 7693291; r931) → set both from one per-pass decision; don't force every pass (multiview branch 4efd827/914177a).
13. **App's const create-info mutated.** In-place `viewMask` write into `VkPipelineRenderingCreateInfo` (i). → deep-copy the `pNext` chain you change.
14. **Right eye stale/black after post-processing.** App blits/copies/resolves with `layerCount=1`. → widen for upgraded↔upgraded transfers (dev 3fbd243); also barriers and clears (i).
15. **Right eye black in compute- or RT-driven passes.** Neither is multiview. → per-layer dispatch / `depth=2` launch with eye from `.z`.
16. **Deferred lighting mono or "doubled".** Lighting FS samples the G-buffer as `sampler2D` (layer 0) in both views. → FS patch: arrayed type + `ViewIndex` layer coordinate (r242, r298 SuperTuxKart).
17. **All textures turn arrayed / invalid SPIR-V in the FS.** `OpTypeImage` edited in place although shared with material textures. → clone the type per stereo variable; retype the whole pointer/sampled-image/load chain.
18. **MSAA samples break after FS patch.** `Arrayed` is `OpTypeImage` word 5; checking word 6 (MS) was the bug. (58ce78e)
19. **Wrong G-buffer layer or wrong texture arrayed.** Binding-number heuristic (bindings 0–4). → track upgraded views through descriptor writes keyed by set layout + binding; or own the shader.
20. **`spirv-val`: capability/extension.** Missing `MultiView` capability, or missing `SPV_KHR_multiview` in a SPIR-V 1.0–1.2 module, or extension emitted before the last capability. (aa42525, d6632fc, 1d071be)
21. **`spirv-val`: decoration placement / Flat.** `BuiltIn ViewIndex` decoration outside the annotation section; integer Input without `Flat` in FS. (9ac851a, fcc8e94)
22. **`spirv-val`: duplicate types.** Re-declared `OpTypeBool` / `vec3` / `int`. → look up before declaring. (265b19a, ef6c688, 66055b7, 53d5310)
23. **Hang at pipeline creation.** Patcher loop `continue` without advancing the word cursor. (96baf05)
24. **Invalid vector select / store after rewrite.** Scalar condition on a vector `OpSelect` (needs SPIR-V 1.4), `vec4` stored to `vec3`. (e816e3e, a18bc17)
25. **Offset never applied in some shaders (i).** Only the final `OpReturn` of the position-writing function is instrumented; early returns or a position written in a callee whose caller is the entry point are missed. → wrap the entry point instead.
26. **Eyes swapped.** Constants named by near-field sign; select order flip-flopped. (01ef4e2, e9710d2, acf50eb, 2dd9f91, 391c452) → name by far-field effect; test with a near object (crossed) and a far one (uncrossed).
27. **Hotkeys "don't work".** Eye constants baked as `OpConstant`; the UBO the hotkeys update is never read. → live parameters via a buffer the shader reads (§4.4).
28. **`projection=parallel` ignored (i).** String option parsed with the boolean reader → default. → parse enums as strings.
29. **Convergence tweak also changes background depth.** `(1 − C·w)` couples screen plane and infinity disparity. → expose dynamic3d's `sep`/`conv` and derive (§4.2).
30. **Skybox pops out of the screen.** Sky drawn at a small radius gets finite-depth disparity. → flatten: constant infinity disparity + `z = w` (2ab68d1, 2e3a559, f04be32).
31. **Black strip at one edge of a procedural sky.** The shifted fullscreen quad no longer covers the viewport. → stretch by `1 + E` (17f61cc).
32. **Sky in front of everything with reversed-Z (i).** `z = w` is near under reversed-Z. → choose 0 or w from the depth compare op.
33. **Only some pipelines in a batch patched (i).** `stereo_CreateGraphicsPipelines` jumps to a label *after* the per-pipeline loop (`goto PIPE_DECISION_CONTINUE`) when a pipeline is not multiview or is a UI candidate, so every later pipeline in the same `vkCreateGraphicsPipelines(count > 1)` call is left unpatched (and, on the UI-candidate path, the VS scan tables leak). → `continue` inside the loop.
34. **Pipelines that can't be patched at all.** Shader-module identifiers (`VK_EXT_shader_module_identifier`), shader-object binaries, pipeline binaries carry no SPIR-V (VKS3D logs identifiers and refuses SO binaries). → refuse or strip the identifier path so the app supplies SPIR-V; the release notes' "delete the shader/pipeline cache" advice is the user-side workaround.
35. **Shader objects / Zink on new drivers: one eye black.** Shader-object paths aren't fully handled (release notes: downgrade driver; env `VKS3D_SHADER_OBJECTS_MONO`). → treat shader objects as a separate integration (dynamic-rendering `viewMask` + patched VS/FS at `vkCreateShadersEXT`).
36. **Multi-device / multi-instance apps (i).** Command hooks use "the first device" (`find_any_device`). → dispatchable-handle → device map.
37. **Crash or black right after resize.** Swapchain recreation path (oldSwapchain reuse, destroy of the reused slot). (f5123da, PR #8/#9)
38. **Overlay tools (MSI Afterburner/RTSS) cause one-eye-black/freeze** (release notes). Overlays inject their own Vulkan layer/draws into the fake swapchain (i). → document; disable overlays when testing.

---

## 10. Config knobs worth exposing

From `vks3d.ini` / `stereo_config_init` — and what to keep if you build your own:

| Knob | VKS3D | Recommendation |
|---|---|---|
| master enable | `enabled` / `STEREO_ENABLED=0` | keep; also a runtime toggle |
| separation, convergence | `separation`, `convergence` (+ `STEREO_*` env) in the §4.1 form | expose dynamic3d's `sep` (screen-width fraction at infinity) and `conv` (screen-plane depth); derive constants |
| step sizes + save | `step_separation`, `step_convergence`, Ctrl+F3–F7 save to the per-game ini | keep, but make the parameters live (§4.4) |
| eye swap | `flip_eyes` | keep |
| output mode | `presentation_mode = sbs | nv3dlib | tab | interlaced | dxgi | dx9 | mono | auto` (several BROKEN) | delegate to `output3d`'s mode list; add LeiaSR via interop |
| multiview | `multiview` (code default 0, shipped ini 1) | in your own code there's nothing to toggle |
| mono UI | `mono_ui` | keep as an escape hatch |
| sky | `sky_flatten`, `sky_extend` | keep both; add a reversed-Z override |
| per-shader skip | env `VKS3D_SKIP_SHADER_PATCHES` (comma list or `*`) | keep, by stable hash; add "force stereo" and "force mono" lists |
| dumps / log | env `VKS3D_DUMP_SPIRV=<dir>`, `STEREO_LOGFILE_PATH=<file|debugview>` (no-op when unset, write-through so crashes keep the tail) | keep both |
| global vs. local ini | `<dll dir>\vks3d.ini` then `<exe dir>\vks3d.ini`, then env | keep the layering |
| real ICD | `STEREO_REAL_ICD` | only for ICD-style injectors |

---

## 11. Debugging workflow

1. **Log the first-frame decisions**: per render pass (eligible? twin created?), per framebuffer (all attachments upgraded? which one wasn't?), per pipeline (stage chosen, classification result, shader hash). VKS3D logs exactly these tags (`FB_ATTACHMENT_CLASSIFY`, `FB_MV_NOT_SELECTED`, `MV_BLOCKED_TRACKED_NONMV_FB`, `PIPELINE_UI_GATE`).
2. **Dump and validate SPIR-V** (`VKS3D_DUMP_SPIRV`; `scripts/debug.cmd` runs `spirv-dis` and `spirv-val` on `-xs.spv`/`+xs.spv` pairs). An invalid patched module often "works" on one vendor and crashes another.
3. **Bisect classification with the skip list**: `*` (nothing patched → must look like the 2D game), then individual hashes.
4. **Separation 0 test**: with separation 0 the patched build must be pixel-identical to mono; anything off-screen or black at 0 is a plumbing bug, not math (the release notes' "Black screen → set separation 0" step).
5. **Edge test**: stretched/black left-right edges = shader not stereoized ("needs code change" in the troubleshooting tree); expanded FoV visible at both eyes' outer edges = it works, increase separation.
6. **One eye black** → §9 #14/#15/#35; **eyes identical** → pass not multiview or shader not patched; **depth inverted** → eye order (#26).
7. Use the Vulkan Samples matrix (`SaschaWillems/Vulkan`) as a regression suite — VKS3D's release notes keep a per-sample status list, which is a good template.

---

## 12. Applying this inside a renderer you control (emulators, engines)

**Emit, don't patch.** If your backend generates SPIR-V (Xenia's and Ryujinx's shader translators, Dolphin's, an engine's material compiler), emit the ViewIndex input and the offset at the position write in the generator. All of §3.2 becomes "use the builder correctly", and the parameters can be a field in a UBO/push constant you already bind.

**Allocate eyes in the render-target cache, not by usage.** Your cache knows which targets are scene colour/depth (two layers) and which are shadow/cubemap/probe/UI (one). Render passes / rendering scopes that write two-layer targets get `viewMask = 0b11`; that same flag picks the pipeline variant (pipelines must match, §2.3/§2.5).

**Mirror every non-draw operation.** Emulators are full of copies, resolves, format reinterpretations and compute passes over render targets (EDRAM resolves on Xbox 360, texture-cache copies and blits on Switch). Each must operate on both layers — usually by looping the layer or widening `layerCount`; compute resolves dispatch with `z = 2` (§2.7). This is the bulk of the work and the main source of "right eye stale".

**Classify by guest semantics where you can.** In an emulator the guest's shaders are D3D/NVN-style: matrix×vector as dot products, constants from buffers — so §5's matrix provenance doesn't apply, but the `dot_count`, Position-input-shape and z/w-use signals do, and you can do better with guest knowledge (viewport/scissor, which constant buffer holds the projection, ortho vs. perspective test on the projection itself if it's identifiable). Keep a per-title override list keyed by **guest** shader hash (stable across host driver/emulator versions), not host SPIR-V hash.

**Layered rendering conflicts.** Guest draws that already render to array layers or cube faces (layered render targets, `Layer`/`ViewportIndex` writes) must stay out of multiview — mirror VKS3D's cube-view exclusion at the target level (i).

**Output** through `output3d` from the 2-layer image (§8.1), LeiaSR via `leiasr-integration` (D3D interop per §8.2 if the weaver is D3D-only).

**OpenGL side (i, general knowledge, not from VKS3D):** OpenGL has the analogous `GL_OVR_multiview`/`GL_OVR_multiview2` (`glFramebufferTextureMultiviewOVR` on a 2-layer texture array, `layout(num_views = 2) in;` and `gl_ViewID_OVR` in GLSL). Vendor support is uneven and it imposes its own restrictions, so for a GL wrapper the two-pass approach is often the pragmatic one; if the target is a Vulkan-capable app, routing it through a Vulkan path (Zink is how VKS3D gets GL titles into 3D — release notes) is the other option.

---

## 13. Reference implementation

**[ThreeDeeJay/VKS3D](https://github.com/ThreeDeeJay/VKS3D)** — "Universal stereoscopic 3D Windows Vulkan driver", GPL-3.0. Studied at `main` r2949 (`82c9522`, 2026-10-20). Nothing in this skill is copied from it; read it for the real thing.

| Path | What to read it for |
|---|---|
| `src/icd_main.c` | ICD entry points, physdev stub list, `stereo_GetDeviceProcAddr` hook list, `vk_icdEnumerateAdapterPhysicalDevices` |
| `src/stereo.c` | real-ICD discovery, INI/env layering (`cfg_float`/`cfg_bool`), `stereo_config_compute_offsets`, device dispatch table |
| `src/instance.c`, `src/device.c` | version cap, forced extensions/features, stereo UBO |
| `src/render_pass.c` | `create_mv_rp`, RP2 twin, depth-only / shadow pass exclusion, `PRESENT_SRC` rewrite |
| `src/framebuffer.c` | per-framebuffer MV selection, `stereo_CmdBeginRenderPass`, `stereo_CmdBeginRendering`, `stereo_CmdTraceRaysKHR` layer copy |
| `src/swapchain.c` | fake swapchain, `stereo_CreateImage` / `stereo_CreateImageView` upgrade rules |
| `src/shader.c` | `do_scan` (provenance), `emit_body` (the formula + sky variants), `spirv_patch_stereo_vertex`, `spirv_patch_stereo_mesh`, `spirv_patch_stereo_fs`, `spirv_patch_stereo_raygen`, `stereo_CreateGraphicsPipelines` (routing + classification), `stereo_CreateShadersEXT` |
| `src/present_alt.c` | `gpu_compose_sc_init`/`gpu_compose_present` (blit SbS), `hotkeys_poll` |
| `src/present_nv3d.cpp` | D3D11 shared texture + D3D12 fence → Vulkan external memory + timeline semaphore |
| `src/tes_inject.c` | the abandoned TCS+TES injection (why not to) |
| `vks3d.ini`, `scripts/debug.cmd` | knobs; dump/validate/skip workflow |
| `tests/` | stale vs. the current formula (see §4.1) |

Branches with relevant unmerged work: `dev` (transfer wrappers that widen blits, FS binding experiments, RTX Remix raygen), `multiview` (stop forcing `viewMask` on every dynamic-rendering pass and every RP2), `spirv-val` (FS retyping validation fixes), `rtx-remix`.

Milestones worth reading as diffs: `5130c69` (DXVK: don't rely on matrix ops), `6571ca2` (usage-based upgrade), `200fe77`/`129d9d1` (shadow pass exclusion), `d57f3b5`/`7693291` (dynamic rendering), `58ce78e` (MSAA operand), `aa42525`/`d6632fc`/`9ac851a`/`fcc8e94` (SPIR-V validity), `2ab68d1`/`2e3a559`/`17f61cc`/`f04be32` (sky), `f5123da` (resize), PR merges #16 (dynamic rendering), #22/#32/#34 (SPIR-V validation), #27 (MV render pass classification), #30 (sky/UI depth), #31/#35 (UI classification).

Related public tools mentioned in its notes: `oneup03/NV3D-Lib` (3D Vision output on modern drivers), DXVK / DXVK-Sarek / DXVK-Remix and MESA Zink (to bring D3D8–11 and OpenGL titles onto Vulkan).

---

## What this skill does NOT cover

- Choosing, tuning and animating separation/convergence, auto-convergence, dynamic crosshair — `dynamic3d` (§4.2 here only maps the parameterizations).
- Output formatting beyond the 2-layer→SbS blit and the D3D interop pattern — `output3d`, `extraoutput3d`.
- Sky-element math beyond the generic flatten/extend — `skybox-stereo-depth`.
- LeiaSR SDK integration — `leiasr-integration`.
- OpenXR/VR (multiview is also the VR path, but swapchain, pose and FoV handling are different).
