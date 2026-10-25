---
name: emulator-stereo
description: Add stereoscopic 3D (side-by-side and LeiaSR weaving) to a console EMULATOR you fork — either by adding geometry stereo inside its GPU emulation (worked case: Xenia-3D, an xenia-canary fork on Direct3D 12, every guest draw rendered once per eye) or by routing and weaving a console's own stereo pair (worked case: an Azahar 3DS fork). Covers choosing the shape before writing code, the vertex shift inside the shader translator (one shader, a per-draw system constant, w == 1 kept at the screen, shadow-map-shaped viewports left mono), what needs a right-eye copy and what must stay single (guest-memory GPU copy, emulated framebuffer memory such as Xenia's EDRAM on the ROV path, resolves and clears, a texture-cache eye bit only for GPU-written pages, per-eye resolution-scale state, CPU readbacks and occlusion/ZPD counts from the left eye only), presenting two front buffers as one double-width guest output (post-AA off, the 10-bit guest output the SR weaver renders black), an on-screen display drawn INTO both eyes because the emulator's own notifications are drawn after weaving, emulator hotkeys that collide with Ctrl+F-keys and the GetKeyState toggle-bit bug, config systems that save the stored value instead of the live one, per-game 3D values, community game patches that may be downloaded but not bundled, keeping the proprietary SR SDK in a separately built bridge DLL behind a C API, weaver teardown on a cleanup thread so exit can't hang with the eye-tracking camera on, measured costs, unattended testing by posting messages to the emulator window, and a checklist for other emulators. Load it when the task is 3D/stereo/SBS/SR in an emulator (Xenia, Azahar/Citra, Ryujinx, shadPS4, Dolphin-style renderers) rather than in a PC game you don't own.
---

# emulator-stereo — stereo 3D inside a console emulator you fork

An emulator is the friendliest place to add stereo: you own the renderer, the shader translator, the presenter and the input loop, and the console's GPU state is fully visible to you. It is also the place where "render it twice" hides the most traps, because the emulated GPU writes into guest memory that the emulated CPU, the texture cache and later draws all read back. This skill is the field notes from two forks:

- **Xenia-3D** — [nautymac/Xenia-3D-SR](https://github.com/nautymac/Xenia-3D-SR) (branch `xenia-3d`; a fork of xenia-canary `c28af1125`, Direct3D 12 backend only). Geometry stereo added inside the GPU emulation: every guest draw is issued once per eye, each eye with its own copy of the emulated framebuffer memory and of the GPU copy of guest memory. Verified by playing WET and Burnout Revenge, side by side and woven on a LeiaSR panel.
- **Azahar-SR** — an Azahar (3DS) fork, not public. The console already renders two eyes; the fork adds a "Simulated Reality" stereo option that renders both eyes at full window size and weaves them, with OpenGL and Vulkan renderers. Verified on a LeiaSR panel (Super Mario 3D Land, Shovel Knight, Rayman 3D, 3D Shinobi III and others).

*Added in the nautymac fork. Every number below was measured on an RTX 4080 Laptop GPU with a 3840x2160 LeiaSR panel.*

Related skills: `dynamic3d` (§4.7 has the image-based auto convergence Xenia-3D uses), `leiasr-integration` (the weaver itself; §B.4 for weaving a Vulkan renderer through D3D11), `output3d` (compose modes), `vulkan-multiview-stereo` (§12: multiview inside an emulator's or engine's Vulkan renderer, worked on a shadPS4 port), `api-proxy-stereo` (when you do NOT own the renderer: a PC game).

---

## 0. Decide the shape first

| The console / renderer | Shape | Worked case |
|---|---|---|
| Already renders a stereo pair (3DS) | Route both eyes into a 2W frame, weave or show SBS. No stereo math. | Azahar-SR (§9) |
| Mono console, emulated framebuffer in one buffer you control (Xenia's EDRAM on the ROV path) | **Per-draw duplication** inside the command processor: draw, switch eye state, draw again | Xenia-3D (§1–§8) |
| Mono console, Vulkan renderer with ordinary host render targets | `VK_KHR_multiview` + a patched pre-raster stage | shadPS4 port — `vulkan-multiview-stereo` §12 |

**Do not ship alternate-eye-per-frame.** Xenia-3D's first milestone rendered the left eye on even guest frames and the right eye on odd ones, to validate the shift and the present path cheaply. It did that job, and it is not a product: in play it showed broken graphics and occasional black frames, because every render-to-texture effect, resolve and feedback buffer is then shared between two eyes that disagree, and each eye only runs at half the game's frame rate. It was replaced by per-draw duplication the same day.

---

## 1. What gets two copies, and what must stay single (Xenia-3D)

The rule: **anything the emulated GPU writes and something later reads must exist per eye; anything the emulated CPU observes must come from exactly one eye.**

| State | Per eye? | How |
|---|---|---|
| Vertex position | shifted per eye | one translated shader; a system constant carries the eye shift (§2) |
| GPU copy of guest memory ("shared memory", 512 MB) | **two buffers** | second committed buffer; every CPU upload copied into both; same resource state and barriers for both; tiled (sparse) shared memory switched off with stereo so neither needs mirrored tile mappings |
| Emulated framebuffer (EDRAM buffer, ROV path) | **two buffers** | `edram_buffers_[2]`; active eye selects buffer, GPU address and descriptor heap start |
| Resolves (EDRAM → guest memory) and the ROV clears they carry | **per eye** | right eye resolves into its own shared memory first, then the left eye as usual |
| Texture cache entries | **only when GPU-written** | an `eye` bit in the texture key, set for the right eye only if the texture's pages were written by the GPU (§4) |
| Resolution-scaled resolve buffers | **per eye** | one scaled-resolve state per eye (buffers, heaps, current range) |
| Front buffer (swap texture) | **per eye** | right one requested under eye 1, from the right eye's memory |
| CPU readbacks of resolves (memory the game's CPU reads) | **left only** | the left eye's resolve does the readback; the right eye's does not |
| Occlusion queries / ZPD counters | **left only** | the right eye's draw gets no counter |
| Draws into square or strip viewports (shadow maps, LUTs) | rendered twice, **not shifted** | eye shift 0 outside the screen-view aspect range |

Cost: about twice the GPU work of the mono emulator and about 0.5 GB more video memory (the second shared-memory buffer plus the second EDRAM buffer).

Requirements Xenia-3D enforces at startup, turning stereo off for the session with a logged reason instead of rendering garbage: bindless resources (the per-eye descriptor tables, §3) and rasterizer-ordered views (stereo forces the ROV path). Stereo on/off is **latched at startup** because the per-eye resources are created then; the output (SBS / SR) can switch live.

---

## 2. The vertex shift inside the shader translator

Put the shift in the translator, at the one point every translated vertex (or domain) shader passes through just before it writes the position output. That covers every game shader with one code path, and the per-eye difference is a constant, not a second shader.

```cpp
// DxbcShaderTranslator::CompleteVertexOrDomainShader(), before the position is written.
// Equivalent HLSL:   if (pos.w != 1) pos.x += xe_stereo.x * (pos.w - xe_stereo.y);
if (stereo::IsEnabled()) {
  a_.OpAdd(temp_y, position.wwww, -LoadSystemConstant(kStereo, stereo_convergence));
  a_.OpMul(temp_y, temp_y, LoadSystemConstant(kStereo, stereo_eye_shift));
  a_.OpNE (temp_x, position.wwww, 1.0f);
  a_.OpMovC(temp_y, temp_x, temp_y, 0.0f);   // w == 1: 2D and full-screen passes stay at the screen
  a_.OpAdd(position.x, position.x, temp_y);
}
```

- **Where the constant lives.** Xenia's system constant buffer had 8 bytes of padding after `param_gen_window_offset`; Xenia-3D puts `float stereo_eye_shift, stereo_convergence` there (`xe_stereo` in the RDEF table and in `xenos_draw.hlsli`). Use existing padding where you can: growing the system constant buffer changes the layout every translated shader was compiled against.
- **Emit it only in stereo sessions.** The shift code is generated only when stereo was on at startup (§1), so a mono session runs the unmodified translation and pays nothing. If your emulator persists *translated* host shaders, key them on the stereo flag.
- **Per-draw eye shift, not per-frame.** `UpdateSystemConstantValues` sets `stereo_eye_shift` for each draw: `±separation/100` NDC for screen views, `0` otherwise. A screen view is decided from the guest viewport (`PA_CL_VPORT_XSCALE/YSCALE`, or the viewport extent when scaling is off): aspect between `stereo_min_viewport_aspect` 1.15 and `stereo_max_viewport_aspect` 2.5. Square shadow maps and strip-shaped targets are then drawn identically for both eyes, which is what keeps shadows from being cast from two light positions.
- **Units.** Separation is percent of the screen width at infinity (`dynamic3d` §1.3; default 3 %); convergence is in the game's own W units (default 1.0). Games differ wildly: WET's close-ups sit around W = 6–13, Burnout Revenge's chase camera settles near 2.6. Ship auto convergence (`dynamic3d` §4.7) rather than a per-game default.

---

## 3. Per-draw duplication in the command processor

```cpp
// D3D12CommandProcessor::IssueDraw (simplified)
SetStereoEye(0);                         // every draw starts on the left eye,
                                         // even if the previous one failed mid-way
// ... primitive processing, pipeline, textures, constants, bindings for eye 0 ...
const uint32_t eye_count = shared_memory_->IsStereo() ? 2 : 1;
for (uint32_t eye = 0; eye < eye_count; ++eye) {
  if (eye) {
    SetStereoEye(1);
    texture_cache_->RequestTextures(used_texture_mask);    // right-eye variants (§4)
    rebind the pipeline if something changed it;
    UpdateSystemConstantValues(...);                        // eye shift for eye 1
    if (!UpdateBindings(...)) { SetStereoEye(0); return false; }
    re-transition the predication buffer if used;
  }
  issue the draw exactly as the mono path does (index buffer, predication, ...);
}
SetStereoEye(0);                         // CPU readbacks and everything else use the left eye
```

`SetStereoEye(eye)` switches the active eye of the shared memory, the render-target (EDRAM) cache and the texture cache, and **invalidates every binding that differs per eye**: the shared-memory/EDRAM root table, the descriptor-index constant buffers for both stages, and the system constants. Missing one of these produces a right eye that silently reads the left eye's memory — no error, just a "right eye equals left eye" region or garbage.

**Why the ROV path.** Xenia has two output paths: host render targets (many real render targets, ownership transfers between them) and rasterizer-ordered views (the whole emulated EDRAM is one UAV buffer and the pixel shader does the blending). Duplicating one buffer is tractable; duplicating the host-RT path's web of render targets and transfers is not. Stereo forces the ROV path and disables itself when the GPU lacks ROVs.

**Why bindless, and the table trick.** With bindless descriptors the per-eye resources become per-eye descriptor *tables* and switching eyes is one root-parameter change. Upstream kept the EDRAM UAV in the general view heap at a fixed offset; Xenia-3D moved it next to the shared-memory views so one table holds everything that differs per eye. The tables are laid out as 4 tables of 3 descriptors — shared memory SRV, shared memory UAV, EDRAM R32 UAV — indexed `(eye * 2 + shared_memory_is_uav) * 3`.

**Uploads.** `UploadRanges` copies every CPU-written range into both shared-memory buffers in the same command list, and both buffers always share one resource state; a transition or UAV barrier is pushed for both.

---

## 4. Guest memory written by the GPU: the texture-cache eye bit

Resolves copy the emulated framebuffer into guest memory, and games then sample that memory as an ordinary texture (render-to-texture, reflections, post-processing chains, the front buffer). After per-eye resolves those pages differ between the eyes, so a texture over them needs a right-eye variant. Textures uploaded by the CPU are identical in both eyes and must **not** be duplicated — that would double the texture cache for nothing.

```cpp
// TextureCache::ApplyActiveEyeToKey: called wherever a binding's key is built for the active eye
key.eye = 0;
if (active_eye_ && key.is_valid &&
    (shared_memory().IsRangeGpuWritten(key.base_page << 12, base_extent) ||
     shared_memory().IsRangeGpuWritten(key.mip_page  << 12, mips_extent)))
  key.eye = 1;                 // load this texture from the right eye's shared memory
```

`IsRangeGpuWritten` reuses the shared memory's existing per-page "valid and GPU-written" bits — the emulator already tracks them for invalidation. Changing the active eye resets the texture bindings so the next draw re-requests them; `RequestSwapTexture` goes through the same key logic, which is how the right eye's front buffer comes from the right eye's memory.

---

## 5. Resolves, clears, readbacks and queries

- **Resolve the right eye first.** `IssueCopy` switches to eye 1, resolves (and clears) the right eye's EDRAM into the right shared memory, switches back, then runs the normal left-eye resolve, **including any CPU readback**. The CPU only ever sees left-eye results, so game logic that reads rendered memory (occlusion tricks, screenshot-to-texture, GPU-computed data) runs once, on one consistent world.
- **Occlusion is counted for the left eye only** (the right eye's draws get no ZPD counter). Counting both would double the sample counts the game reads.
- **Front buffer.** At swap, request the right eye's swap texture under eye 1 *before* the left one, so the command processor ends the swap on the left eye.

---

## 6. Presenting two eyes as one guest output

- Apply the emulator's own front-buffer processing (Xenia's `apply_gamma`) **per eye**, into two eye-sized outputs, then copy both into one guest output twice as wide. Keep the display aspect of one eye: on a normal screen that is half-width SBS; a 3D TV or the weaver stretches it back.
- **Turn off post-processing that samples across pixels** on the combined image (Xenia: FXAA) — it blurs across the seam between the eyes.
- **SR weaving.** The presenter weaves after its effect pass. Xenia's guest output is `R10G10B10A2`, and the LeiaSR weaver produced a **black** image from it (also from a stable copy of it). Fix: let the normal effect pass draw the SBS image into the 8-bit `B8G8R8A8` back buffer's output rectangle, `CopyTextureRegion` that rectangle into an 8-bit weaver input texture, and weave it back into the same rectangle with a viewport set to it. Hold the switchable lens on only on frames that were actually woven (`leiasr-integration` §A.10).
- **Tearing** is very visible on SR and 3D displays. Xenia allows tearing by default; Xenia-3D keeps that default (it is the low-latency choice on fast GPUs) and adds a 3D-menu toggle that stops passing `DXGI_PRESENT_ALLOW_TEARING` live. Turning it back on needs a swap chain created with tearing allowed, i.e. the next start.

---

## 7. Resolution scaling and measured cost

Upstream's draw-resolution scaling keeps resolution-scaled copies of resolved memory in tiled buffers. Each eye needs its own: Xenia-3D keeps one scaled-resolve state per eye (buffers, heaps, buffer indices, current range), selected by the texture cache's active eye, and commits memory for both eyes. With that in place the 1x restriction of the first version was lifted; the 3D menu offers 1x/2x/3x for the next start.

WET (one gameplay spot, ROV path in all rows):

| Mode | Frame rate | GPU load |
|---|---|---|
| 2D, 1x | 30 | ~34 % |
| 3D (SR woven), 1x | 30 | ~39 % |
| 2D, 1x, community 60 FPS patch | 60 | ~40 % |
| 3D (SR woven), 1x, 60 FPS patch | steady 60 | ~57 % |
| 3D, 2x | 30 | ~65 % |
| 3D, 2x, 60 FPS patch | ~50 | 99 % |
| 3D, 3x | 28 | 99 % |

CPU time was about the same in 2D and 3D: the emulator's CPU side, not the second eye, is what limits most games at 1x. The first run at a new scale spends a lot of CPU compiling pipelines; the second run is normal.

---

## 8. Controls, overlay and settings — the emulator-side traps

**On-screen display: draw it into the eyes, before composing.** The emulator's own notification overlay (Xenia's ImGui) is drawn by the presenter **after** weaving, so on an SR panel it lands on top of the woven image as unreadable 2D. Xenia-3D renders its OSD (separation, convergence, auto convergence, output, tearing, fps) as GDI text into an `R10G10B10A2` texel image, uploads it only when the text changes, and copies it into **both** eye outputs at the same position (zero disparity, at the screen plane) before they are combined. It then reads correctly in SBS and when woven. Shown for 3 s after a change; Ctrl+F1 keeps it on.

**Hotkeys collide with the emulator's own F-keys.** Xenia binds almost every plain F-key: F1 opens its FAQ and F2 its build-commit web page, F4 traces a GPU frame, F5 clears the GPU caches, F6 opens a display-config dialog, F11/F12 toggle fullscreen and take a screenshot. Its key handler does not check Ctrl, so Ctrl+F-keys fire them too. Xenia-3D polls its Ctrl+F1…F7 hotkeys on the GPU thread at each guest swap with `GetAsyncKeyState`, only when its own window is in front, and consumes the "pressed since last call" bits even when they don't apply (so a press in another window cannot fire later). The window's `OnKeyDown` returns early for Ctrl+F1…F12.

**`GetKeyState(VK_CONTROL) & 0x1` is the toggle bit, not "held".** Upstream's `Win32Window::HandleKeyboard` built its modifier flags from the low bit, which flips on every press, so Ctrl was seen on every other press — and the Ctrl+F-key early return above missed every other press. The user-visible symptom: Ctrl+F2 (auto convergence) opened Xenia's build-commit web page every second time. Use `& 0x8000` for Shift/Ctrl/Alt; only Caps Lock is a toggle.

**Settings that actually save.** Xenia's `config::SaveConfig()` writes each setting's *stored config value*, not the live `cvars::` variable, so assigning the cvar and saving writes nothing new. Xenia-3D's `stereo::PersistSetting(name, value)` finds the setting in `cvar::ConfigVars` and calls `OverrideConfigValue` before saving (the `OVERRIDE_*` macros only compile in the file that defines the setting). Applied to the output mode, tearing and resolution scale.

**Per-game values.** Ctrl+F7 writes `[Stereo]` separation, convergence and auto convergence into the title's own config file (`config/<title ID>.config.toml`, which Xenia already reads when that game starts); a menu item saves them as the global default instead. Upstream read the game file with `LoadConfigValue`, which made the game's values the *stored global* values, so the next global save (from a menu, even a window resize) leaked one game's settings into every game. `LoadGameConfigValue` fixes it — verified: a game file with 4.5 % loaded while the global file stayed at 3 %.

**Community game patches: download, don't bundle.** Xenia Canary applies `<title ID> - <name>.patch.toml` files from its `patches` folder; the community repository (xenia-canary/game-patches, including ~150 60 FPS patches) has no licence, so Xenia-3D does not redistribute it. A 3D-menu command downloads it at runtime (GitHub tree API, compares git blob SHA-1 with local files so later updates fetch only new or changed files, keeps patches the user enabled, never deletes local files) and reloads the patch database so new files apply at the next game start.

---

## 9. Azahar-SR: a console that already renders two eyes

The 3DS renders a left and a right image natively, so there is no stereo math — the work is layout, presentation and weaving.

- **A new stereo option value.** `Settings::StereoRenderOption::SimulatedReality = 7`, appended so existing config values keep their meaning. In it each eye is rendered at the **full window size** and the frame is twice as wide as the window (`render_full_stereo`-style layout); touch input lands in the left eye's half, which matches the window. The software renderer maps the mode to plain side by side.
- **Screenshots and video dumps** of an SR session use the plain 2D layout (`framebuffer_layout.cpp` maps the mode to Off): the woven picture only means something on the panel, and the SR layout has no room for a second eye in a dump.
- **OpenGL: copy, then weave.** The presented frame is a renderbuffer owned by the render context; FBOs are not shared between the render and presentation contexts, but textures and renderbuffers are. So the presenter copies the mailbox renderbuffer into a texture with `glCopyImageSubData` (no FBO needed), neutralises state the weaver's full-window pass must not inherit (blend, scissor, depth, stencil, cull, sRGB, colour mask; viewport = window), weaves, and leaves the context as Azahar expects (program 0, VAO 0, framebuffer 0). If weaving is unavailable, the same frame is blitted as half SBS.
- **Vulkan: through D3D11.** The SR SDK has no Vulkan weaver, so the Vulkan presenter copies the 2W frame into a D3D11 texture shared with Vulkan, the SDK's DX11 weaver weaves it, and the result is copied back into the swapchain image. A D3D11 fence imported as a Vulkan timeline semaphore orders the two APIs on the GPU. The recipe is `leiasr-integration` §B.4. It was worth it: Super Mario 3D Land (with a 60 FPS patch) ran at **60 fps, full speed, ~40 % GPU at 9x internal resolution on Vulkan**, while the OpenGL path already ran below full speed at 6x and at 23 fps at 9x.
- **Tear the weaver down on its own thread.** On exit, Azahar stops emulation from the emulation thread while the GUI thread waits for it. Destroying the DX11 weaver there deadlocked: the weaver's destroy waits for its processing threads, which talk to the window's (GUI) thread. The visible symptom was a windowless `azahar.exe` that never exited — **with the eye-tracking camera still on**, because the process still held the SR context. Fix in the bridge: retired weavers go to a detached cleanup thread that destroys them and then releases the SR context ("context released (eye tracking off)"); a later restart creates a new context. The OpenGL path drops (does not destroy) its weaver per new window, so there the camera stays on until the process exits.
- **Texture filters can break an emulated 2D layer.** 3D Shinobi III draws an emulated Mega Drive picture as a texture; every texture filter (ScaleForce, Anime4K) garbled it. Per-game `texture_filter = 0` fixed it. When a game embeds another console's output, test with filters off first.

---

## 10. The SR bridge DLL and the licence boundary

Both forks keep the SR SDK out of the emulator binary: a small DLL, built separately with its own CMake, links the proprietary SDK, delay-loads the SR runtime, and exports a **plain C API** (`xenia3d_sr.dll`: initialize / weave / set lens / shutdown; `azahar_sr.dll`: GL weave, DX11 setup, DX11 weave, stop). The emulator `LoadLibrary`s it at runtime; when the DLL or the runtime is missing the output stays side by side, so one build serves SR panels and ordinary 3D displays.

- **GPL emulators (Azahar):** the SDK is not GPL-compatible, so it stays out of the GPL binary entirely.
- **BSD emulators (Xenia):** linking would be allowed, but the SDK is not in the repository and Xenia builds with the static CRT — a C API keeps C++ exceptions, STL types and allocators from crossing between two runtimes.

Ship the bridge next to the emulator exe and say in the README that it is optional.

---

## 11. Testing an emulator build unattended

- **Post messages to the emulator window; never use global synthetic input.** `keybd_event`/`SendInput` go to whatever window is in front, and Windows refuses `SetForegroundWindow` while the user works in another app — test keystrokes then landed in the user's editor. Xenia-3D adds a debug setting, `hid_keyboard_ignore_focus`, so its keyboard-as-controller driver accepts key messages posted to the window without focus; menus are driven with `WM_MENUCOMMAND` by position (Xenia's menus use `MNS_NOTIFYBYPOS`).
- **Never touch the user's own emulator instance.** Kill only your test instance by PID or path, and don't run a test build while the user's emulator is running.
- **Measure, don't eyeball.** Log the auto-convergence measurements (`stereo_log_auto_convergence`), log the guest frame rate (`stereo_log_frame_rate`), and check disparity with block matching on dumped frames (e.g. WET's intro measured 32–36 px at convergence 1, i.e. everything near infinity — the cue that convergence was far too small).
- **Azahar specifics.** It rewrites `qt-config.ini` at startup, and a key with `key\default=true` ignores its value: set the value **and** `default=false`. Its log is buffered and only flushed on a graceful exit, so a killed instance shows a truncated log. Key messages posted to the GL render child window work without focus.

---

## 12. Checklist for another emulator

1. **Shape** (§0). Native stereo console → layout + weave. Mono console → per-draw duplication if the emulated framebuffer is one buffer you can twin; otherwise multiview into the Vulkan renderer (`vulkan-multiview-stereo` §12 — one stereo predicate evaluated identically at image creation, pipeline key and shader specialisation).
2. **Shift in the translator**, last pre-raster stage, `x += shift * (w - conv)`, `w == 1` untouched; per-draw shift 0 for non-screen viewports.
3. **List every GPU-written resource** that a later draw, resolve, copy or the CPU reads. Each one is per eye, mirrored, or left-only — decide explicitly (§1 table). Guest-memory GPU copies and emulated framebuffer memory are always per eye.
4. **Texture cache:** an eye bit only for GPU-written ranges.
5. **CPU-visible results from one eye**: readbacks, queries, counters.
6. **Present** two eye outputs into one 2W output; post-AA off; check the weaver's input format.
7. **OSD into the eyes**, not into the emulator's post-weave overlay.
8. **Hotkeys** that don't collide with the emulator's own; `& 0x8000` for modifiers.
9. **Settings** that persist through the emulator's config system; per-game values in its per-game file.
10. **Auto convergence** from the eye images (`dynamic3d` §4.7), capture labelled with the values it was drawn with.
11. **SR bridge DLL** with a C API, lens only while woven, teardown off the emulation thread.
12. **Unattended tests** by window messages only.

---

## 13. Pitfalls — symptom → cause → fix

| Symptom | Cause | Fix |
|---|---|---|
| Broken graphics, occasional black frames, half frame rate per eye | alternate-eye-per-frame stereo: render-to-texture and feedback buffers shared between disagreeing eyes | per-draw duplication with per-eye guest memory (§0, §1) |
| Right eye shows the left eye's picture in reflections / post effects | right eye sampled a texture over GPU-written pages from the left eye's memory | texture-key eye bit for GPU-written ranges (§4) |
| Right eye partly wrong after switching eyes | a binding that differs per eye was not invalidated (root table, descriptor-index CBVs, system constants) | invalidate all of them in the eye switch (§3) |
| Doubled shadows / shadows from two light positions | shadow-map draws shifted per eye | shift only screen-view viewports (aspect 1.15–2.5) (§2) |
| Game logic reads doubled or inconsistent rendered data | CPU readback or occlusion counters from both eyes | left eye only (§5) |
| Blurred seam in the middle of the SBS image | post-AA (FXAA) over the combined image | turn it off with stereo (§6) |
| Woven SR output is black | weaver input was the 10-bit `R10G10B10A2` guest output | weave from an 8-bit copy of the composed back-buffer rectangle (§6) |
| OSD/notifications unreadable on the SR panel | emulator overlay drawn after weaving | draw your OSD into both eye images before composing (§8) |
| Ctrl+F-key works every other time / opens the emulator's own function | `GetKeyState & 0x1` (toggle bit) for modifiers; plain F-keys bound by the emulator | `& 0x8000`; skip Ctrl+F-keys in the emulator's key handler (§8) |
| Saved separation/convergence not written | config system saves the stored value, not the live cvar | `OverrideConfigValue` before saving (§8) |
| One game's 3D values appear in every game | per-game file loaded as the global stored value | load it as the game value (§8) |
| Exit hangs, process stays without a window, eye-tracking camera on | weaver destroyed on the emulation thread while the GUI thread waits | destroy weavers on a cleanup thread, then release the SR context (§9) |
| Garbled 2D layer with texture filters on | filter applied to an emulated other-console picture | per-game `texture_filter = 0` (§9) |
| Test keystrokes typed into another app | global `keybd_event` while the emulator is not in front | post messages to the emulator window (§11) |

---

## What this skill does NOT cover

- The weaver API itself, SDK vendoring, delay-load and lens policy — `leiasr-integration`.
- Multiview into a Vulkan renderer — `vulkan-multiview-stereo`.
- The auto-convergence estimator and controller in detail — `dynamic3d` §4.7.
- Compose modes beyond SBS and SR (interlaced, anaglyph, frame-packed) — `output3d`, `extraoutput3d`.
- Stereo for PC games you don't have the source of — `api-proxy-stereo` and `wrapper-injection`, and for flat 2D games `flat2d-layered-depth`.
- Xenia's Vulkan backend: Xenia-3D is Direct3D 12 only.
