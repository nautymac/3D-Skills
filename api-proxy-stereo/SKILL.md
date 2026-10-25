---
name: api-proxy-stereo
description: Inject generic geometry stereo 3D (geo-11 / 3D Vision automatic style, no per-game shader fixes) into PC games you do NOT have the source of, through a drop-in API proxy DLL placed next to the game exe — opengl32.dll (OpenGL, x86 + x64), vulkan-1.dll (Vulkan, and D3D9/D3D11 games routed through DXVK), d3d12.dll (DirectX 12, DXIL and DXBC SM 5.0). Covers export forwarding and the loading traps (pin the DLL because DXVK frees vulkan-1; GL 1.1 entry points handed out by wglGetProcAddress bypass you; program-binary caches), the last-pre-raster-stage patch `pos.x += eye*sep*(w-conv)` with w==1 kept at screen depth in GLSL / SPIR-V / DXIL / DXBC, where the stereo parameters live (Vulkan push constant at maxPushConstantsSize-16 appended to every pipeline layout; D3D12 root constants and the SM 5.0 lesson — cb14+ cannot be declared and DX11-style engines fill every per-stage CBV table, so carve b13), which render targets get right-eye twins (window-aspect vs non-square rule, strips, mip pyramids, views), per-draw duplication on each API (GL clone FBOs; Vulkan image/view/framebuffer/descriptor-set twins and a tee into a secondary with state replay; D3D12 doubled descriptor heaps and twice-run compute), buffer twins for tile/light lists and atomic counters, what duplication cannot reach (vendor upscalers like DLSS, deferred back-projection ghosting, path tracing — and the path-traced SBS-in-one-frame fix), presenting from inside a process you don't own (virtual back buffer against half-drawn frames, a full state guard before your own draws, upscale windows on the game window's thread, DPI and display-mode handling by import-table patches only), hotkey swallowing, and a measure-don't-trust verification and debugging toolkit (disparity from frame dumps, bisect switches, per-pass L/R dumps, desktop-duplication flicker capture, a hang watchdog that cannot deadlock the game). Load it when the task says wrapper, proxy DLL, injector, "make this game 3D", geo-11-like for OpenGL/Vulkan/DX12, DXVK stereo, Helldivers/UE4/Unity game in 3D without source, or when debugging a stereo wrapper (one eye wrong, grey or ghosted right eye, flicker, black screen, device lost, hang).
---

# api-proxy-stereo — generic geometry stereo through a drop-in API proxy DLL

This skill is for the case none of the other skills assume: **you don't own the renderer.** The game is a shipped binary; you put one DLL next to its exe, the game loads it instead of the system graphics DLL, and every frame comes out in stereo — with no per-game shader fixes, only rules that work across engines. It is the playbook distilled from three such wrappers, each verified in real games on an SR (lenticular) panel and in side-by-side:

| Wrapper | Proxy | Verified in |
|---|---|---|
| **OGL3D** — [nautymac/OGL3D-SR](https://github.com/nautymac/OGL3D-SR) (`main` = 1.0.0 line, `flato3d` = later generic fixes) | `opengl32.dll`, x64 + x86 | Curse of the Dead Gods, Amnesia: The Dark Descent (deferred + fixed-function + immediate mode), Darkest Dungeon, Broken Age; on `flato3d` also WineD3D-translated D3D9/D3D11 games (see `wrapper-injection`, `flat2d-layered-depth`) |
| **VK3D** | `vulkan-1.dll`, x64 | Lara Croft GO (Unity 5.4 D3D9 → DXVK), Little Nightmares (UE4 D3D11 → DXVK, menu/title), Quake II RTX (path traced, per-game fix) |
| **DX12_3D** | `d3d12.dll`, x64 | Helldivers 2 (Stingray, DXBC SM 5.0, ship and missions), Ghostrunner 2 (UE4.27 DXIL, geometry) |

VK3D and DX12_3D are not public; they are described here without links. Code references below point at OGL3D-SR where the same idea exists there.

*Added in the nautymac fork; every claim was checked in a running game. Where a finding is only known for one API or one game, it says so.*

Relationship to the other skills: the stereo math is `dynamic3d`'s clip-space parameterization (`x += sep * (w - conv)`); the compose/output modes are `output3d`; handing the side-by-side frame to an SR weaver from inside someone else's process (32-bit runtime probes, DPI threads, Vulkan → D3D11 interop) is `leiasr-integration`; image-based auto-convergence for wrappers that cannot read the game's depth is in `dynamic3d`; getting the DLL loaded at all (loader proxies, forwarding shims, static-import patches, WineD3D for D3D8–11) is `wrapper-injection`, and a game that stops responding is `windows-hang-diagnosis`; flat 2D games (no depth reaches the GPU) are `flat2d-layered-depth`; emulators you fork are `emulator-stereo`; single-pass multiview injection for Vulkan is the alternative in `vulkan-multiview-stereo`.

---

## 0. Before you start

**Anti-cheat: don't.** A proxy DLL patches shaders, hooks the API and rewrites command streams — exactly what anti-cheat looks for. Never deploy into games with kernel anti-cheat or competitive online play; a ban is permanent and lands on the user's account, not yours. Test on single-player/offline titles. (The Helldivers 2 tests below ran only with the owner's explicit acceptance of that risk; they are not evidence that it is safe. Never accept a game's EULA on the user's behalf either.) DX12_3D's `[System] passthrough = 1` (forward everything, hook nothing) is the cheap first test of whether a game accepts the DLL at all.

**Cost.** Geometry stereo draws the right eye again. Expect roughly half the frame rate: Little Nightmares' UE4 menu went ~95 → ~45 fps (61 stereo, 40 mono passes per frame), Helldivers 2 ran ~36 fps at 4K per eye (1132 draws per frame, 635 stereo). That is what 3D Vision cost too. The one large exception is path tracing (§5.3).

**Pick the route by what the exe really renders with.** Imports of `opengl32.dll` / `vulkan-1.dll` / `d3d12.dll` / `d3d11.dll` / `d3d9.dll` are a first guess, not proof (a Unity build can import opengl32 and render D3D11). If the wrapper never writes its log, the game is not using that API. D3D11 games are best served by geo-11/3Dmigoto; D3D9 and D3D11 games can also be translated to Vulkan with DXVK (`d3d9.dll`, or `d3d11.dll` + `dxgi.dll`) and get VK3D's generic engine — verified in Lara Croft GO and Little Nightmares, experimental. Routing D3D8–11 to OpenGL through WineD3D is covered in `wrapper-injection`.

---

## 1. The proxy DLL

### 1.1 Forward everything, implement a little

Generate the export list from the system DLL instead of writing it: OGL3D's `tools/gen_exports.py` reads `System32\opengl32.dll` and `SysWOW64\opengl32.dll` and writes a `.def` plus forwarding stubs (`src/gen/`), with the exports you implement listed in `src/exports_impl.txt`. Load the real DLL by **full system path**, never by name (the loader would find you again). VK3D does the same for `vulkan-1.dll`.

Then hook at the API's own dispatch point, not with inline patches:

- **OpenGL:** implement the core exports you need; for everything else return your hook from `wglGetProcAddress`.
- **Vulkan:** answer `vkGetInstanceProcAddr` / `vkGetDeviceProcAddr` with your functions. VK3D generates a wrapper for all 278 `vkCmd*` functions from `vulkan_core.h` (`tools/gen_cmds.py`) and hand-writes only the ones with stereo logic (render passes, binds, draws, copies, barriers).
- **D3D12:** patch the vtables of the objects the device creates. Compute the slot numbers from the SDK's C interface declarations (`offsetof` in the `...Vtbl` structs) instead of counting by hand, and keep the originals per vtable.

### 1.2 Loading traps

| Trap | What happened | Fix |
|---|---|---|
| **Your DLL gets unloaded** | DXVK loads `vulkan-1.dll` per DXGI factory and frees it again; UE4 creates a throwaway factory. VK3D's exception handler then ran inside an unloaded DLL (WER: `vulkan-1.dll_unloaded`). | Pin yourself in `DllMain`: `GetModuleHandleExW(GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS \| GET_MODULE_HANDLE_EX_FLAG_PIN, (LPCWSTR)&DllMain, &h)`. |
| **GL 1.1 calls bypass you** | SDL + glad (AGS 3.6 games) fetch even `glDrawArrays`, `glClear`, `glBindTexture` through `wglGetProcAddress`, and NVIDIA answers with the driver's own entry points: the draws never reached the wrapper and the right eye stayed black. | In `wglGetProcAddress`, return **your own export** for every GL 1.1 name you implement (OGL3D `flato3d`, `kOwnGl11` in `src/proxy.cpp`). |
| **Cached program binaries skip your shader patch** | Games restore linked programs with `glProgramBinary`. | Refuse binaries: report `GL_NUM_PROGRAM_BINARY_FORMATS = 0` **and** return nothing for `GL_PROGRAM_BINARY_FORMATS`. Faking only the count crashed `nvoglv64`: the game sized its buffer from our 0 and the driver still wrote its list. |
| **Third-party DLLs in the process call your exports** | The SR SDK's GL weaver imports `opengl32.dll`, so its own GL calls go through your hooks. | Weave with no game context active (`t_ctx = nullptr` in OGL3D) and a thread-local "own shaders" flag so the weaver's shaders are not patched; restore state afterwards. |
| **The game never loads your DLL name** | Static imports, SteamStub-wrapped exes, overlays mapping the system DLL first. | A loader proxy (`version.dll`, or a `winmm.dll` forwarding shim) that redirects the loads — see `wrapper-injection` §2. |

---

## 2. The shader patch

### 2.1 One formula, last pre-rasterization stage

Patch the **last** stage before rasterization (GS > DS/TES > VS) and add, after the game's own position output:

```
pos.x += eye * separation * (pos.w - convergence)     // eye = -1 left, +1 right, 0 mono
// pos.w == 1  →  2D, HUD, full-screen passes: leave it (or shift by a constant HUD depth)
```

This is `dynamic3d`'s clip-space form, so its calibration, auto-convergence and sky rules apply unchanged. `w == 1` is the generic HUD detector that makes "no per-game fixes" work: orthographic UI, post-process triangles and blits stay at screen depth; perspective geometry gets depth. DX12_3D and VK3D expose `hud_depth` (`w == 1` content shifted by `hud * w`); OGL3D keeps it at the screen (`mono_w1 = 1`). Things the formula alone cannot do — screen-space passes, path tracing, flat 2D — are §5 and the other skills.

### 2.2 GLSL (OpenGL)

Rename the game's `main` and append a new one (OGL3D `src/inject.cpp`):

```glsl
// game source with "void main(" renamed to "void s3d_main0("
uniform vec4 s3d_P; // x = eye, y = separation, z = convergence, w = keep w==1 mono
void main() {
    s3d_main0();
    if (s3d_P.x != 0.0 && abs(gl_Position.w - 1.0) > 0.00001)
        gl_Position.x += s3d_P.x * s3d_P.y * (gl_Position.w - s3d_P.z);
}
```

- Find `main` with a regex on a copy of the source whose comments are blanked to spaces (same length, so offsets still line up). Renaming is robust: early `return`s in the game's `main` are harmless.
- Set `s3d_P` per draw from the wrapper; uniform locations are per program.
- **No shader = no injection point.** Fixed-function, ARB vertex programs and GLSL that reads the built-in matrices get the offset through a **per-eye projection matrix** instead (compatibility contexts; `fixed_function = 1`). Immediate mode (`glBegin`/`glEnd`) is recorded into a display list while it runs for the left eye and replayed for the right; game display lists are replayed per eye too.
- If you generate identifiers from game identifiers, never produce `__`: GLSL reserves names containing two underscores, and a LOVE game's outputs starting with `_` failed to compile once prefixed.

### 2.3 SPIR-V (Vulkan)

- The parameters are a `vec4` **push constant at `min(maxPushConstantsSize, 256) - 16`** (240 on NVIDIA). Append it to every pipeline layout: extend the game's existing ranges that cover the pre-raster stages (a stage may appear in only one range) or add a new range; if the game already uses those bytes, leave that layout alone. Push the eye value before each draw.
- Patch with a real SPIR-V module editor (find or add the push-constant block, add the member, `OpAccessChain` to it after the position store) and validate patched modules with `spirv-val` in a test suite (VK3D builds SPIRV-Tools for that).
- **DXVK passes shaders inline** (`VkShaderModuleCreateInfo` chained into the pipeline stage, maintenance5), so `vkCreateShaderModule` never sees them: patch and dump at **pipeline creation**.
- Per-game shader replacement works by hash: VK3D's fix files map an FNV-1a 64 hash of the original SPIR-V to a replacement module (that is how the Quake II RTX fix in §5.3 is delivered).

### 2.4 DXIL and DXBC (Direct3D 12)

- **DXIL:** round-trip through DXC (`dxcompiler.dll` + `dxil.dll` shipped next to the proxy): disassemble → edit the LLVM IR text → assemble → write your own container hash. Find `SV_Position` through `!dx.entryPoints` → signatures → outputs; after the block where both `x` and `w` were stored, load the parameter cbuffer and store `x` again. Name your new SSA values (`%dx3d.*`) so the numbering of the unnamed ones stays valid.
- **DXBC (SM 4/5):** patch the token stream. Redirect every write to the position output into a new temp, and before every `ret`/`retc` (VS/DS) and `emit` (GS) write the shifted copy out; keep the temp itself unshifted so a fall-through `retc` doesn't accumulate. `dcl_temps` grows, the parameter cbuffer is declared, the container is re-hashed. Both paths were verified pixel-exact against a test renderer before touching a game.

### 2.5 Where the D3D12 parameters go — the SM 5.0 lesson

DX12_3D adds 4 root constants `{sep, conv, hud, 0}`. Choosing the register looked trivial and was not (Helldivers 2):

1. **Shader model 5.0 can only reach space 0 and cannot declare `cb14` or above** (fxc limit). Put the constants at `b31` and the driver silently reads 0 — and some PSOs fail with `E_INVALIDARG`. So search **`b13` down to `b0`**, free for all stages first.
2. **DX11-style engines leave no register free.** Stingray's root signatures bind `cbv 0..31, srv 0..31, uav 0..31, samplers 0..31` per stage through per-stage descriptor tables. Look for a register free for the one stage you patch (VS/DS/GS; overlap is checked per visibility).
3. **Else carve.** Split that stage's CBV range around `[Generic] steal_register` (default **b13**, the 3Dmigoto convention — almost never used) and bind your root constants there with that stage's visibility. Carve only from a **stage-specific** table, never from an all-stage table: that would take the register from pixel and compute shaders you cannot check (tried; broke compute).
4. Else your own register space (`space 0x7D3D`), which only DXIL/SM 5.1 shaders can address.

Respect the 64-DWORD root-signature budget, record in each PSO which slot its patched shader reads, and log any shader that itself declares the stolen register ("already uses the parameter register") so the user can pick another. About 10 PSOs per Helldivers 2 launch still failed with the patched shader and fell back to the original; they were not in the frames that mattered and were not analysed.

---

## 3. Which render targets get a right eye

Only targets that hold the **view** need a twin; shadow maps, LUTs and atlases must stay mono (twinning them costs memory and breaks nothing visibly, until it does).

| Rule | Where | Verdict |
|---|---|---|
| **Window aspect** (within 3 %, at least 64 px wide) + the default framebuffer | OGL3D | Fine for GL games, which render at window size. |
| **Non-square, not a strip**: 2D, one layer, attachment or storage usage, not square, neither side more than `max_target_aspect` (4) times the other | VK3D default `stereo_targets = nonsquare`, DX12_3D | Needed for UE4: it renders at the game's own resolution (3840×2400, 16:10) on a 16:9 panel, which the aspect rule missed entirely. Square = shadow map / LUT / atlas. |
| **Strips are never stereo** | OGL3D `flato3d` | Unity's colour-grading LUT is 1024×32; twinning it made the right eye red. |
| **Square UAV mip pyramids** (HZB / depth pyramid built from scene depth) | DX12_3D `stereo_depth_pyramids` | Square but per-eye content — occlusion culling of the right eye reads them. |
| **Views follow their resource** | all | A view of a stereo texture gets a twin view of the twin (GL `glTextureView` from WineD3D, Vulkan image views, D3D12 descriptors). A GL clone that must be viewable needs immutable storage like the original. |

Two rules for every twin:

- **Start it with the left content.** The game may have drawn into or uploaded the texture before it became stereo; a fresh twin made Amnesia's right eye read garbage (cyan banded lighting). GL: `glCopyImageSubData` per level; Vulkan: twins made while the original is already in use start `UNDEFINED`, so bring them into the original's tracked layout.
- **Mirror every copy, clear, resolve and barrier** onto the twins, or the right-eye post chain reads stale data.

---

## 4. Per-draw duplication

Every draw into a stereo target runs twice: the game's call for the left eye, then again with the right-eye targets, the right-eye views of everything it samples, and `eye = +1`.

### 4.1 OpenGL

A shadow FBO per game FBO whose attachments are the clones; the default framebuffer's right eye is an off-screen FBO. While drawing the right eye, every bound texture that has a clone is swapped for it, so post-processing chains stay per eye. **Track the active texture unit yourself** (`glActiveTexture`) — the swap code changes it and must restore exactly what the game had. Mirror `glCopyImageSubData` (WineD3D implements D3D `CopyResource` with it), blits and clears onto the clones.

### 4.2 Vulkan (VK3D)

- **Twins** for every stereo image (made when its memory is bound), its views, the framebuffers using them, and the **descriptor sets** that reference them (copy the whole set, then repoint the stereo entries; decode update templates and push descriptors too).
- **Tee instead of re-record.** While the game records a render pass that draws into stereo images, every command also goes into a **secondary command buffer** (from a shadow pool per game pool) with twins substituted. When the pass ends, begin the same pass again on the twin attachments with `VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS` and execute the secondary. Dynamic rendering the same way.
- **Secondaries inherit no state.** Remember every `vkCmdSet*`/`vkCmdBind*` and replay the latest values into the secondary first. Generate this: VK3D's generator wraps all 278 `vkCmd*`, never tees queries/timestamps/transform feedback/conditional rendering (they would count twice), and doesn't keep events or pointer-carrying structs as replayable state.
- **Deep-copy `pNext` chains** of everything you replay later — known structures only, drop the rest. A remembered state struct whose `pNext` pointed into the game's freed memory crashed the NVIDIA driver during replay.
- **Rebind descriptor sets with the bound pipeline's layout.** Little Nightmares lost the device in gameplay until the right-eye rebind used the layout of the pipeline actually bound (every set that pipeline uses is compatible with it); found by bisecting with `[Debug] disable = …` (§8.2).
- **Compute** dispatches that write stereo storage images run again with the twin sets; copies, clears and barriers on stereo images are repeated on the twins. The swapchain images get twins too, composed side by side at present.
- **Hide what you cannot follow.** In generic mode VK3D removes `VK_EXT_descriptor_heap`, `VK_EXT_descriptor_buffer`, `VK_EXT_graphics_pipeline_library`, `VK_EXT_shader_object`, `VK_EXT_shader_module_identifier`, the device-generated-commands extensions, `VK_KHR_pipeline_binary` and `VK_EXT_nested_command_buffer` from `vkEnumerateDeviceExtensionProperties`. DXVK falls back without them.
- **Log what you were replaying** in the crash handler ("while replaying vkCmdSet…") — a crash inside a replayed command points at the driver, not at you.

### 4.3 Direct3D 12 (DX12_3D)

- **Descriptor heaps are created twice as large**: slot `i + count` holds the right-eye view of slot `i` (the twin, or the same descriptor for mono resources); mirror `CopyDescriptors*` into the right halves and give the right eye's tables the offset.
- Draws into stereo targets are recorded twice (left constants/targets/tables, then right-half RTV/DSV, right-half tables, right constants). Barriers, clears, discards, copies and resolves are mirrored; render passes are emulated; bundles and `ExecuteIndirect` are handled.
- **Compute dispatches that write a stereo UAV run twice** — decide from the UAV registers the shader declares (reflection), not by guessing.

### 4.4 Buffers written by duplicated passes — tile lists, counters, indirect args

The Helldivers 2 mission bug: staircase-edged black blocks. Tiled light culling appends into lists through manual 4-byte atomic counters (230,400-entry lists, 48-byte indirect argument records). Duplicate those dispatches over a **mono** buffer and both eyes append into the same list: it overflows, tiles go unprocessed, and the right eye reuses the left eye's light lists (wrong lighting around the character).

- **Fix: right-eye buffer twins.** Any buffer an eye-duplicated dispatch writes gets a twin, created at Present with the queue idle and zeroed; its right-half views are re-created from the recorded view descs. **Cascade:** a pass that reads a twinned resource and writes buffers runs twice as well, and `ExecuteIndirect` takes the twin argument buffer for the right eye. Games rotate their list buffers, so queue twins on every duplicated dispatch, not only the first sighting. Cap the memory (`stereo_buffer_max_mb`).
- **Dead ends:** restoring a counter between the eyes and leaving the right eye's value behind — the CPU reads that counter back, and the game crashed at load. Twins keep the game's own buffer left-only, which is what its CPU side expects.

---

## 5. What duplication cannot reach

### 5.1 Work done outside your hooks — vendor upscalers

With DLSS on, Helldivers 2's right eye turned flat grey from the first 4K pass on. The scene rendered at 1080p in both eyes (all 1080p passes identical L/R), then DLSS (NGX, inside the NVIDIA driver) upscaled only the left texture; the right twin was never written, and sharpen/bloom inherited the grey. The per-pass L/R dumper (§8.3) proved it in one capture. **Workaround:** native resolution or the engine's own upscaler (Helldivers 2's engine upscaler works). **Proper fix (not done):** evaluate the upscaler per eye — separate feature instance and history per eye, by hooking NGX `EvaluateFeature` / `ffxDispatch` / `xessD3D12Execute` — as geo-11 does for DLSS. Dynamic resolution makes the symptom scene-dependent: check upscaling settings before blaming a stereo fix.

### 5.2 Screen-space passes at the mono position — back-projection ghosting

Deferred lighting, screen-space shadows, SSAO, fog and decals reconstruct world positions from depth with the **mono** camera matrices. Geometry is shifted per eye, the reconstruction is not, so the effect lands offset by the eye shift.

- **Ghostrunner 2 (UE4.27, DX12):** per-pass dumps showed scene colour clean through the base pass; the "ghost" appeared when a light was applied through its screen-space shadow mask.
- **Little Nightmares (UE4 D3D11 via DXVK + VK3D):** each SBS half showed the scene doubled by ~30–150 px. Not the other eye: the offset was constant over depth (disparity is not), the mono frame was clean, and it grew with convergence, roughly `sep * (conv / depth - 1)`.
- **The fix is per-eye matrices** ("stage 2"): per-eye copies of projection/inverse-projection/view-projection in the constant buffers, or regex patches of lighting/shadow shaders (geo-11's matrix analyser / `fix_BackProjection*` plus per-game `ShaderRegex`). DX12_3D has an experimental `[Matrices]` redirect (camera matrices found by reflection names, rewritten per eye inside the shader). In Helldivers 2 it gave only a small measured gain once light lists were per eye (§4.4) and coincided with a one-tile dark strip, so it ships off. Lower separation and convergence hide the ghost; otherwise route the game to geo-11, whose community fixes solve it per game.

### 5.3 Path tracing — trace each half from its own eye

Quake II RTX is fully path traced; vertex shifts do nothing. The working fix (VK3D per-game profile, built from the GPL Q2RTX sources) **traces the left half of the frame from the left eye and the right half from the right eye**, off-axis cameras converging at the convergence distance: the ray count stays that of a mono frame. Measured at 4K: mono 31.2 fps, SBS 30.5, SR-woven 29.5–31 — against the frame-halving of sequential-frame wrappers. Details that mattered: the primary-ray, reflection/refraction and gradient-reprojection shaders all pick the eye by screen half; HUD pictures are drawn into both halves; the first-person gun is traced with cameras much closer together (it is ~4 units from the camera). Rebuild the unmodified sources first and require a byte-identical match with the shipped shader pack before compiling your patch.

### 5.4 Flat content

Videos, comic panels, CPU-composited frames and orthographic 2D games have no depth to shift (`w == 1` everywhere). That is `flat2d-layered-depth` (layered depth rules) or AI depth — not something this engine can fix.

---

## 6. Presenting from inside a process you don't own

### 6.1 Virtual back buffer (OpenGL)

Broken Age flickered: half-drawn frames (background layer only, menu half drawn) and un-woven left eyes flashed on screen. Probes inside the DLL saw only clean composed images; a magenta marker painted between "left eye taken" and "composition" showed up on the **display**. Windows/the compositor sometimes showed the back buffer while it was still being drawn — with vsync on.

Fix (`[Display] virtual_backbuffer = 1`, default): the game's framebuffer 0 is an off-screen FBO, and the real back buffer only ever receives the finished frame. That means mapping 0 → your FBO in every binding (and every binding you restore), `GL_BACK` → `GL_COLOR_ATTACHMENT0` in `glDrawBuffer`/`glReadBuffer`, reporting 0 / `GL_BACK` back through `glGetIntegerv`, refusing attachments to framebuffer 0, and rebinding when the game deletes the bound FBO. Not possible on a multisampled default framebuffer (fall back and log it). Measured with a desktop-duplication capture: gameplay 29 flashes in 40 s → 0. Also: some games call `SwapBuffers` on frames where they drew nothing — reuse the previous left eye instead of composing garbage.

### 6.2 Never let the game's frame-end state reach your draws

Broken Age ended some shots with blend still on as `(GL_DST_COLOR, GL_ONE_MINUS_SRC_ALPHA)`. The wrapper only saved and restored state, so the weaver's full-screen pass was multiplied into the previous back buffer every frame: black within frames (additive leftovers: very bright), until a click changed the game's last draw. The eyes were bright the whole time — only the presentation was black.

Before **any** draw of yours (weave, compose, overlay) **disable** blend, depth, stencil, cull, scissor, `GL_FRAMEBUFFER_SRGB`, logic op, alpha-to-coverage, rasterizer discard, clip distances (and alpha test in compatibility contexts), and set fill mode and colour mask; then restore. The same guard must force `glClipControl(GL_LOWER_LEFT, GL_NEGATIVE_ONE_TO_ONE)`: WineD3D keeps the D3D origin (`GL_UPPER_LEFT`, `GL_ZERO_TO_ONE`) and the woven picture came out upside down (OGL3D `GlStateGuard` in `src/stereo.cpp`, `flato3d`). Log changes of the game's frame-end blend state at a verbose log level; it finds the next such bug in one run.

### 6.3 Your own presentation window

When the game must keep a low resolution (readable UI) but the panel needs native pixels, OGL3D's upscale mode presents in its own owned, topmost, no-activate window:

- From OpenGL, go through **D3D11** (`WGL_NV_DX_interop2`): a second *GL* window presented from a DPI-unaware process crashed in NVIDIA's DXGI-based GL present.
- **Create the window on the game window's thread.** `WM_NCHITTEST → HTTRANSPARENT` passes the mouse only to windows of the **same thread**, and a render thread may have no message pump at all (Samorost 2 under WineD3D: "not responding", clicks ignored, because the window lived on WineD3D's render thread). Install a `WH_GETMESSAGE` hook on the game window's thread, post it a registered message, create the window there — without waiting (that thread may be waiting for yours) — and use asynchronous calls (`ShowWindowAsync`, `SWP_ASYNCWINDOWPOS`, posted `WM_CLOSE`) across threads.
- Upscale the eyes to the window size before handing them on; the SR weaver fills only an input-view-sized area (`leiasr-integration`).

### 6.4 DPI and display modes

- A DPI-unaware game on a 200 % 4K panel renders at half resolution and gets stretched — fatal for a lenticular weave. `[Display] dpi_aware = 1` calls `SetProcessDpiAwarenessContext(PER_MONITOR_AWARE_V2)` from `DllMain`, early enough to win; with its own resolution setting raised to the panel's, Amnesia then rendered native 4K.
- To keep the panel at its native mode while the game "changes" resolution: **patch the import tables of the game's own modules** (exe and DLLs in its folder) for `ChangeDisplaySettings*`/`EnumDisplaySettings*`, report the blocked change as done, and never let a faked mode reach the GPU driver. **MinHook inline hooks on those user32 functions corrupted user32's internals** (crash in an ntdll `memcpy`): user32 calls them internally and the driver calls them too. Import-table patches touch only the callers you mean.
- Third-party DLLs that read window/monitor geometry on the game's DPI-unaware threads (the SR runtime subclasses the window) get their user32 geometry imports patched to run per-monitor aware — same technique (OGL3D `PatchSrModules`, `src/present.cpp`).

### 6.5 Overlay and readbacks

- Draw your OSD at composition into **both** halves (never into the eye copies, which feed the next frame's estimators), and verify it reaches the woven output — DX12_3D's overlay was rasterized but never drawn until a user reported missing values.
- **Readbacks never wait.** Fence them and skip a frame if not done: mapping a PBO "that should be ready two frames later" blocked forever once the GPU queue sat behind a D3D11 presentation. In Vulkan, a zero-batch submit with a fence works as the marker.

---

## 7. Input

- **Swallow your hotkeys.** Games react to F-keys whatever the modifiers: Door Kickers: Action Squad toggled its frame statistics on every Ctrl+F5. Install a `WH_GETMESSAGE` hook on the game window's thread that turns your hotkeys' `WM_KEYDOWN`/`WM_KEYUP`/`WM_SYSKEY*` and keyboard `WM_INPUT` into `WM_NULL` (and eats the matching key-up), and read your keys with `GetAsyncKeyState` while the game window is in front (`[Keys] swallow = 1`; OGL3D `KeyMsgHook`, `flato3d`).
- Overlays still see them: Steam takes F12 for screenshots, so Ctrl+F12 frame dumps also produce a Steam screenshot.
- Use Ctrl+F-keys with a small, fixed map shared across your wrappers (separation F3/F4, convergence F5/F6, save F7, auto-convergence F2, values F1, dump F12, toggle Ctrl+T, weave toggle Ctrl+Alt+W) so users don't relearn them per API.

---

## 8. Verify and debug — measure, don't trust

### 8.1 Disparity numbers, not impressions

Dump both eyes **before weaving** (Ctrl+F12 / `dump_frames`) and block-match them: OGL3D logs a 12×6 grid of horizontal disparity (right − left, px; + = behind the screen) at quarter resolution, with `!` where the eyes do not match at all. Read it like a depth map: HUD 0, far content positive, near content negative, same object same number. A "Stereo 3D is Active" OSD proves nothing: helifax's OpenGL wrapper in its depth mode reported active and produced **pixel-identical eyes**. Leak test: `mono_eye = left|right` (DX12_3D) gives one eye no shift; it must then match a 2D frame exactly.

### 8.2 Bisect the engine, not the game

Make every part of the engine switchable from the ini and bisect in one session:

- VK3D `[Debug] disable = tee, dispatch, sets, vs, barriers, copies, replay` (found the Little Nightmares device-lost in a few runs).
- DX12_3D `disable = draws, dispatch, patch, twins, barriers, copies`, plus `mono_eye`, `right_first`, `left_scope` (shift the left eye only in depth-only or colour passes), `dup_range` / `dup_dispatch_range` (only draws/dispatches *a..b-1* get a right eye) and `dup_steps` cycled with a hotkey, each step dumping a frame.
- OGL3D **hunting**: Ctrl+F9 walks the shaders drawn on screen (the selected one is hidden), Ctrl+F8 sets it to stereo / mono / skip, Ctrl+F7 saves `[ShaderOverride] <program hash> = …` — the per-game escape hatch without writing shaders.

### 8.3 Per-pass L/R dumps

When one eye is wrong somewhere in a long frame, capture each pass's render target **for both eyes** (and compute UAV outputs) in one frame and find the first pass where they diverge wrongly (DX12_3D Ctrl+F10: copies RT0 into scratch textures inside the game's command list at every render-target change, writes quarter-size BMPs after Present). That pinpointed both the DLSS grey eye (§5.1) and the shadow-mask ghost (§5.2) in one capture each.

### 8.4 What is actually on the screen

In-process probes can't see presentation bugs (§6.1). Record the display with DXGI Desktop Duplication and flag frames that differ from both neighbours while the neighbours agree (A-B-A): [`tools/flickercap/flickercap.cpp`](https://github.com/nautymac/OGL3D-SR/blob/main/tools/flickercap/flickercap.cpp) writes a CSV per desktop update and an image sheet per flash. For one-off checks, capture the screen from a **per-monitor-DPI-aware** process; a DPI-unaware capture is scaled and hides exactly the bugs you are looking for. `[Debug] mark_backbuffer` (magenta between eye capture and composition) answers "was this shown before composition?" in one frame.

### 8.5 Crashes and hangs

- **Crash log:** a vectored handler that logs code, faulting module + offset and a stack of return addresses into image code. In a wrapper, a crash inside the driver usually has a cause in your replay or patch; log which replayed state was active. D3D12: enable DRED (`[Debug] dred = 1`) for device-lost breadcrumbs. A lost device on a laptop can need a reboot ("GPU is lost") — reboot before concluding a crash repeats.
- **Hang watchdog:** after 4 s without a frame, log where the render thread is ("HANG?"). While the thread is suspended, only `GetThreadContext` and copy raw stack words; resume it; *then* resolve names (`VirtualQuery` + `K32GetMappedFileNameW`). The first version allocated and looked up modules while the thread was suspended — if it was suspended inside the heap or loader lock, the watchdog deadlocked the game it was diagnosing.
- **Buffered logs lie** about where a process died; flush or log unbuffered while hunting crashes. The `ntdll` access violation that WER reports after a crash in a hooked DLL is usually fallout, not the cause.
- 32-bit games need 32-bit tools (dumps from a SysWOW64 shell). For hang investigations (recording a hung process, lock owners, the WineD3D thread-alert hang on Windows 11 24H2) see `windows-hang-diagnosis`; for 32-bit SR runtime probes eaten by Mono, `leiasr-integration`.

**Case study — someone else's wrapper.** helifax's OpenGL wrapper v12.3 crashed Curse of the Dead Gods at start-up whenever shader stereo injection was on. A first-chance vectored handler in a shim logged `0xC0000005` executing address `0x0` with `stack[0]` = a return address inside `ogl3dv.dll`; disassembling there showed `call [slot]` with an int and four floats — `glUniform4f`. The wrapper resolves GL entry points lazily into a table, but one call site lacked the NULL check and ran first. Fix without its source: an `opengl32.dll` shim (the real wrapper renamed `ogl3dv.dll`) that forwards all exports and, on the first current context, fills all 27 slots — resolving names through **System32's** `opengl32.dll`, never through the wrapper, whose hooks would call themselves. Traps on the way: its `Stereo3D = false` did not disable the injection (useless as a bisect switch), and NVAPI error -137 in its log was cosmetic.

### 8.6 Automating tests without hurting the user

- **Never use global synthetic input** (`keybd_event`, `SendInput`) on a machine someone is using: Windows refuses `SetForegroundWindow`, and the keys land in whatever is in front (an editor, in practice). Post `WM_KEYDOWN`/`WM_KEYUP` to the game window. Games reading raw input ignore posted keys; some ignore `SetCursorPos` (use relative `mouse_event` only when the game is guaranteed in front).
- **Back up saves before an automated run and restore them after.** "New Game" overwrites autosaves; know which menu items are safe.
- Keep the display awake (`SetThreadExecutionState`) during unattended runs.
- When someone watches remotely, output side-by-side, not a lenticular weave (`leiasr-integration`).

---

## 9. Pitfalls — symptom → cause → fix

| Symptom | Cause | Fix |
|---|---|---|
| Right eye black in an SDL/glad GL game | GL 1.1 functions fetched through `wglGetProcAddress` got driver pointers | Return your own exports for implemented GL 1.1 names (§1.2) |
| Crash in `nvoglv64` at start-up | `GL_NUM_PROGRAM_BINARY_FORMATS` faked to 0 but the format list still returned | Fake both (§1.2) |
| Crash in an unloaded `vulkan-1.dll` (DXVK) | DXVK freed the proxy with a DXGI factory | Pin the DLL (§1.2) |
| Right eye garbage / cyan bands | Twin created empty | Copy the left content into new twins (§3) |
| Right eye red in a Unity game | Colour-grading LUT strip twinned | Strips (aspect > 4) are never stereo (§3) |
| No stereo at all in a UE4 game | Targets at the game's own 16:10 resolution failed the window-aspect rule | Non-square rule (§3) |
| Device lost only in gameplay (Vulkan) | Right-eye sets rebound with the wrong layout | Rebind with the bound pipeline's layout (§4.2) |
| Crash in the driver during replay | Remembered state with a dangling `pNext` | Deep-copy known `pNext` structs, drop others (§4.2) |
| Black staircase blocks, wrong light near characters (D3D12) | Tile/light lists and counters appended by both eyes into one buffer | Buffer twins + cascade + twin indirect args (§4.4) |
| Some PSOs fail `E_INVALIDARG`, or the shift reads 0 (SM 5.0) | Parameters at `cb14+` | `b13..b0`, carve b13 from a stage table (§2.5) |
| Right eye flat grey from the first full-resolution pass | Vendor upscaler (DLSS) outside duplication | Engine upscaler / native; per-eye upscaler is the real fix (§5.1) |
| Ghost / shadows offset by the eye shift, grows with convergence | Screen-space passes reconstruct with mono matrices | Per-eye matrices or per-game fixes (§5.2) |
| Path-traced game flat | Vertex shift has no effect on rays | Trace each half from its eye (§5.3) |
| Half-drawn or un-woven frames flash, probes see nothing | Compositor showing the back buffer mid-frame | Virtual back buffer (§6.1); verify with desktop duplication (§8.4) |
| Screen goes black or white for a whole shot; eyes are fine | Game's frame-end blend state applied to your weave | Full state guard (§6.2) |
| Woven image upside down (WineD3D) | `glClipControl(GL_UPPER_LEFT…)` left on | Force lower-left in the guard (§6.2) |
| Game "not responding", clicks ignored, in upscale mode | Presentation window created on a pump-less render thread | Create it on the game window's thread (§6.3) |
| Crash in ntdll after hooking display-mode functions | Inline hooks on user32 | Import-table patches of the game's modules only (§6.4) |
| Ctrl+F5 toggles a game feature | Game ignores modifiers | Swallow hotkeys with a `WH_GETMESSAGE` hook (§7) |
| Watchdog freezes the game it watches | Allocation/module lookup while the thread was suspended | Copy raw words while suspended, resolve after resume (§8.5) |
| "Stereo active" but depth looks flat | Eyes identical | Measure disparity from frame dumps (§8.1) |

---

## 10. Config knobs worth exposing

`[Stereo]` `separation` (percent of `max_separation`, a clip-space shift at infinity), `convergence` (number or `auto`), `hud_depth`, `swap_eyes`, `output` (`auto` = SR weave when available, else `sbs`; `tab`). `[Generic]`/`[Targets]` `stereo_targets` (`nonsquare`/`aspect`), `max_target_aspect`, `aspect_tolerance`, `min_width`, D3D12's `steal_register`, `double_dispatch`, `stereo_buffers`, `stereo_buffer_cascade`, `stereo_depth_pyramids`. `[Display]` `dpi_aware`, `virtual_backbuffer`, `upscale`, `swap_interval` (some games turn vsync off, which tears a woven picture). `[Keys]` with `swallow`. `[Debug]` `log_level`, `log_crashes`, `dump_frames`, `dump_shaders`, `stats_every`, `disable`, plus the API-specific bisect keys of §8.2. Reload the ini while the game runs where you can — A/B tests without restarts are worth the code.

---

## 11. Reference implementations

| Project | What to read |
|---|---|
| [nautymac/OGL3D-SR](https://github.com/nautymac/OGL3D-SR) `main` | `src/inject.cpp` (GLSL rewrite), `src/stereo.cpp` (targets, clones, per-draw duplication, virtual back buffer, `GlStateGuard`, disparity grid, hang watchdog), `src/present.cpp` (upscale window, import-table patches, SR DPI patches), `src/proxy.cpp` + `tools/gen_exports.py` (forwarding), `tools/flickercap/` |
| [nautymac/OGL3D-SR](https://github.com/nautymac/OGL3D-SR/tree/flato3d) `flato3d` | Later generic fixes: own GL 1.1 exports from `wglGetProcAddress`, texture views and multisample clones, strip rule, `glClipControl` in the guard, hotkey swallowing, presentation window on the game window's thread |
| VK3D (not public) | Vulkan generic engine described in §2.3, §3, §4.2; Quake II RTX profile (§5.3) |
| DX12_3D (not public) | D3D12 engine described in §2.4–2.5, §4.3–4.4, §5.1–5.2, per-pass dumper |
| [helifax/OGL-3DVision-Wrapper](https://github.com/helifax/OGL-3DVision-Wrapper) | The 3D Vision-era OpenGL wrapper; studied for design, case study in §8.5 |

---

## What this skill does NOT cover

- **Handing the frame to an SR weaver** (runtime loading, x86 probe vs Mono, lens state, Vulkan → D3D11 interop, weaving inside 3Dmigoto): `leiasr-integration`.
- **Auto-convergence from the two eye images** (block matching on shrunk eyes, the aperture-problem fix, the controller): `dynamic3d`.
- **Output modes and the compose shader:** `output3d` / `extraoutput3d`.
- **Getting the DLL loaded** (loader proxies, forwarding shims, static-import patches) and the **WineD3D route** for D3D8–11 into OpenGL: `wrapper-injection`. **Flat 2D games:** `flat2d-layered-depth`. **Hang investigations:** `windows-hang-diagnosis`.
- **Emulators you can modify** (per-draw duplication inside GPU emulation): `emulator-stereo`; single-pass multiview: `vulkan-multiview-stereo`.
- **D3D11 games:** geo-11 / 3Dmigoto and its community fixes are the mature path; this skill's engines do not cover native D3D11 (only via DXVK or WineD3D).
- **Per-eye vendor upscalers, mesh shaders, DXR in D3D12, per-game rule files:** not built yet.
