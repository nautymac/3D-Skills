---
name: leiasr-integration
description: Integrate the LeiaSR autostereoscopic display weaver (Simulated Reality SDK) into a game or renderer that already produces a side-by-side stereo image. Covers the SR runtime contract (SbS-packed input texture, weaver hand-off, init ordering), shipping a binary that runs on machines without the SR runtime installed (delay-load + SEH preflight + LoadLibraryW gate), upscaling from a low render resolution to the SR panel's native resolution via the weaver, OS window-handle plumbing (SDL / GLFW / Win32), per-monitor-DPI-awareness (the weave must land 1:1 on physical panel pixels — on a >100%-scaled display a non-DPI-aware host gets its window/weave virtualized and stretched, collapsing the 3D; set it before window creation, plus the SR-service one-shot-resize correction), vendoring the SDK itself as a git submodule (bo3b/SR-lib, api_expansion branch — a public repo) whose CMake package (SRLib::SDK / SRLib::SR / srlib_apply_delayload) replaces the hand-rolled SDK paths and delay-load list, its SRInterfaceDX9/DX11/DX12/OGL wrapper plus switchable-lens-hint and polled head/eye tracking, its nested per-architecture layout, and the CI credential plumbing (read-access PAT + global insteadOf rewrite) needed only if you mirror it into a private repo of your own, the switchable lenticular lens as global display state you must hand back when you stop weaving, and graceful fallback to plain Side-by-Side when SR isn't available. Also covers diagnosing the failure mode where output is flat mono while every SDK call reports success, and ghost reduction in two layers: tuning the weaver's OWN anti-crosstalk (mode Off/Static/Dynamic, strength as a multiple of the panel's measured factors, and the weaving contrast) — absent from the IDX*Weaver1 interfaces but reachable through the runtime's IWeaverSettings1 via query<>(), which SR-lib's SR.hpp wraps as SetCrosstalk() plus raw-SDK helpers, with tested starting values and the ABI caveat of a hand-declared interface — and why the weaver's cancellation still leaves visible ghosting on high-contrast game content (its correction clips against the ends of the range), plus the sRGB-vs-linear space rule that decides whether the compose-shader fix helps or hurts (and why setShaderSRGBConversion(false,false) on the display-encoded UNORM content almost every game produces silently under-cancels and makes the anti-crosstalk Strength control look inert, with no colour symptom) - that shader fix itself lives in the output3d skill. Also covers hosting the weaver in a process you don't own (a proxy DLL such as dxgi.dll / opengl32.dll / d3d9 hooks, next to another wrapper such as 3DMigoto/geo-11 or TriDef): the 32-bit runtime's Virtual PC probe (illegal instruction 0F 3F 07 0B) that a host's vectored exception handler — Mono in 32-bit Unity games — swallows, hanging or killing the game the moment the weaver is created, and the first-VEH fix; never creating throw-away devices/windows inside another wrapper; finding the real swap chain/device inside wrapper objects; sRGB back buffers (typeless copy, matching input/output views); DPI-unaware games (IAT-patched SR geometry calls, upscaled views); GL state the game leaves on at frame end (blend, clip control); weaver teardown on a thread another thread is waiting for; a stale lens after a crash (restart the SR Service); field data that the per-eye width also weaves correctly on runtime 1.34.10. Backend-neutral entry points are stable; per-backend sections exist for OpenGL, DirectX 12, DirectX 11, Vulkan (through a D3D11 shared texture and a D3D11 fence imported as a timeline semaphore — the SDK has no Vulkan weaver) and Direct3D 9.
---

# LeiaSR autostereoscopic display integration

This skill captures everything you need to **hand a side-by-side stereo image off to a LeiaSR (Simulated Reality) autostereoscopic display** and have the weaver produce a glasses-free 3D image. It is *backend-neutral about how you got the stereo image* — that's the host engine's problem — and picks up at:

> *"I have a GL / DX / Vulkan texture containing a side-by-side stereo pair (LEFT eye in the left half, RIGHT eye in the right half). Make it 3D on a Leia panel."*

The skill is organized so other graphics-API integrations can be added without restructuring:

1. **Shared concerns** (Part A) — apply to every backend: getting the SDK into your tree at all (start at A.0), the SbS-packed input contract, the SR runtime lifecycle, surviving on machines without the SR runtime (the delay-load + SEH dance), upscaling from low render resolutions, window-handle plumbing.
2. **Per-backend specifics** (Part B) — what changes between GL / DX / Vulkan: which weaver type to create, which texture-handle type to pass, where the weaver writes its output. **OpenGL** (B.1) is the worked example, with **DirectX 12** (B.2), **DirectX 11** (B.3), **Vulkan** (B.4, through D3D11 interop) and **Direct3D 9** (B.5) alongside it; other backends can append their own section using the same structure.
3. **Hosting the weaver in a process you don't own** (A.13) — when the weave runs inside a proxy DLL or a hook in somebody else's game (32-bit Unity games, other stereo wrappers, DPI-unaware games, emulators): the failure modes that never show up when you own the host. *Added in the nautymac fork from the geo-11_SR SRWeave, OGL3D, VK3D, DX12_3D, Xenia-3D, Azahar-SR and TriDef SRWeave integrations, all weaving on a SpatialLabs panel (SR Platform 1.34.10).*

A reference implementation lives in the [RingRacers-3D](https://github.com/oneup03/RingRacers-3D) fork ([`src/hwr2/stereo_leiasr.hpp`](https://github.com/oneup03/RingRacers-3D/blob/master/src/hwr2/stereo_leiasr.hpp) / [`stereo_leiasr.cpp`](https://github.com/oneup03/RingRacers-3D/blob/master/src/hwr2/stereo_leiasr.cpp) + the `src/CMakeLists.txt` delay-load block) — when in doubt, that's the canonical source for the OpenGL patterns below. Other public integrations referenced throughout:

| Backend | Project | Files |
|---|---|---|
| OpenGL (MSVC host) | [RingRacers-3D](https://github.com/oneup03/RingRacers-3D) | `src/hwr2/stereo_leiasr.{hpp,cpp}`, `src/i_video_common.cpp` |
| OpenGL (MinGW host + MSVC shim) | [Kart-Public-3D](https://github.com/oneup03/Kart-Public-3D), [perfect_dark_3D](https://github.com/oneup03/perfect_dark_3D) | `leiasr_shim/leiasr_shim.cpp` + `src/r_stereo_leiasr.c` / `port/src/stereo_leiasr.c` |
| DirectX 12 (RT64 / N64 recomp ports) | [rt64-3D](https://github.com/oneup03/rt64-3D) (`Stereo3D` branch), consumed by [Goemon64Recomp3D](https://github.com/oneup03/Goemon64Recomp3D) and [BanjoRecomp3D](https://github.com/oneup03/BanjoRecomp3D) | [`src/render/rt64_leiasr_weaver.cpp`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/render/rt64_leiasr_weaver.cpp) |
| DirectX 12 (Aurora / WebGPU-on-D3D12) | [aurora3D](https://github.com/oneup03/aurora3D) (`Stereo3D-rebased` branch), consumed by [dusklight3D](https://github.com/oneup03/dusklight3D) | [`lib/webgpu/leiasr.cpp`](https://github.com/oneup03/aurora3D/blob/Stereo3D-rebased/lib/webgpu/leiasr.cpp), [`lib/window.cpp`](https://github.com/oneup03/aurora3D/blob/Stereo3D-rebased/lib/window.cpp) |
| DirectX 12 + DirectX 11 (UE injection) | [UU3D](https://github.com/oneup03/UU3D) (Unreal Universal 3D Mod, `flat3d-afw` branch) | [`src/mods/vr/flat3d/Flat3DCompositorD3D12.cpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DCompositorD3D12.cpp), [`Flat3DCompositorD3D11.cpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DCompositorD3D11.cpp) |
| DirectX 11 (OpenVR driver) | [VRto3D](https://github.com/oneup03/VRto3D) | [`vrto3d/src/presenter/leiasr_presenter.cpp`](https://github.com/oneup03/VRto3D/blob/main/vrto3d/src/presenter/leiasr_presenter.cpp) |
| SDK packaging | [bo3b/SR-lib](https://github.com/bo3b/SR-lib) (`api_expansion` branch) | `CMakeLists.txt`, `SR.hpp` / `SR.cpp` |
| DirectX 11, x64 + x86 (`dxgi.dll` proxy next to 3DMigoto/geo-11) | [geo-11_SR](https://github.com/nautymac/geo-11_SR) (`master`) | [`SRWeave/src/srweave_geo11.cpp`](https://github.com/nautymac/geo-11_SR/blob/master/SRWeave/src/srweave_geo11.cpp) — raw SDK `CreateDX11Weaver`, the Virtual PC probe handler (A.13.1), factory hooks (A.13.2), sRGB views (A.13.3) |
| OpenGL in an `opengl32.dll` proxy; DX11 for its upscale mode | [OGL3D-SR](https://github.com/nautymac/OGL3D-SR) (`main`; `flato3d` branch adds the clip-control reset) | [`src/srout.cpp`](https://github.com/nautymac/OGL3D-SR/blob/main/src/srout.cpp), [`src/stereo.cpp`](https://github.com/nautymac/OGL3D-SR/blob/main/src/stereo.cpp) (`GlStateGuard`), [`src/present.cpp`](https://github.com/nautymac/OGL3D-SR/blob/main/src/present.cpp) (`PatchSrModules`) |
| Direct3D 9 + DirectX 11 hooks behind TriDef 3D | [tridef3d-win11-injector](https://github.com/nautymac/tridef3d-win11-injector) (`master`) | [`srweave/src/dx9.cpp`](https://github.com/nautymac/tridef3d-win11-injector/blob/master/srweave/src/dx9.cpp), [`srweave/src/dx11.cpp`](https://github.com/nautymac/tridef3d-win11-injector/blob/master/srweave/src/dx11.cpp) |
| DirectX 12 in an emulator (Xbox 360) | [Xenia-3D-SR](https://github.com/nautymac/Xenia-3D-SR) (`xenia-3d` branch) | [`tools/xenia3d_sr/xenia3d_sr.cc`](https://github.com/nautymac/Xenia-3D-SR/blob/xenia-3d/tools/xenia3d_sr/xenia3d_sr.cc) (bridge DLL), [`src/xenia/ui/d3d12/d3d12_presenter.cc`](https://github.com/nautymac/Xenia-3D-SR/blob/xenia-3d/src/xenia/ui/d3d12/d3d12_presenter.cc) |

Unpublished integrations quoted below (no public source): **VK3D** (`vulkan-1.dll` proxy, Vulkan → D3D11 interop, B.4), **DX12_3D** (`d3d12.dll` proxy, DX12 weaver) and **Azahar-SR** (3DS emulator, GL weaver and Vulkan → D3D11 interop, B.4).

---

# Part A — Backend-neutral

## A.0 Step 0 — vendor the SR SDK (do this before anything else)

Nothing below builds until the SDK is in the tree. The SR SDK isn't on any
public package index, so the working convention is a **git submodule** — and you
want the **`api_expansion` branch**, which carries a CMake package and a DX12 /
lens-hint / tracking wrapper that `master` does not:

```sh
git submodule add -b api_expansion https://github.com/bo3b/SR-lib.git libs/SR-lib
git submodule update --init libs/SR-lib
```

> **`bo3b/SR-lib` is public.** An anonymous clone works, and CI needs no
> credentials for it — plain `submodules: recursive` on `actions/checkout` is
> enough. (An earlier revision of this skill asserted it was private. That was
> wrong, and it cost a consumer a PAT-plumbing detour it never needed.) The PAT
> recipe in B.1's *CI considerations* still applies if you mirror the SDK into a
> private repo of your own, and its two failure modes are worth reading anyway —
> they bite any private submodule.

### Use the CMake package — don't hand-roll the SDK paths

`api_expansion` ships a `CMakeLists.txt` whose entire purpose is that consumers
stop rewriting the same block. Three lines replace it all:

```cmake
add_subdirectory(libs/SR-lib)

if(SRLIB_FOUND)
    target_link_libraries(MyExe PRIVATE SRLib::SR)  # or SRLib::SDK, see below
    srlib_apply_delayload(MyExe)                    # A.3.1, done for you
endif()
```

| Target | What you get |
|---|---|
| `SRLib::SDK` | SDK + third_party include dirs, and every import library |
| `SRLib::SR` | the above **plus** `SR.cpp`'s `SRInterface*` wrapper (A.6) |
| `srlib_apply_delayload(<tgt>)` | the complete `/DELAYLOAD` list + `delayimp` |
| `SRLIB_FOUND` | false on non-Windows, or if the submodule wasn't checked out |

Link `SRLib::SR` if you want the wrapper (almost always). Link `SRLib::SDK` only
if you're talking to `SR::` types directly and deliberately reimplementing what
the wrapper does.

**Why this matters more than saved keystrokes:** the delay-load list is the part
integrations get wrong, and getting it wrong means the `.exe` refuses to launch
on any machine without the SR runtime (A.3.1). Centralising it means an SDK that
adds a DLL fixes every consumer at once. `srlib_apply_delayload` also silences
LNK4199 for DLLs your particular consumer doesn't import — expected, since the
list is deliberately complete rather than tailored.

The package locates the SDK itself: it globs `SR-SDK-*` for the version
directory (so a version bump doesn't break you), picks `x64`/`x32` from
`CMAKE_SIZEOF_VOID_P`, applies the `32` suffix the 32-bit SR libs carry, and
self-heals a cached `SRLIB_SDKROOT` that no longer resolves. You should not need
to set anything.

### What `SRLib::SR` already implements — check here before writing your own

`SR.hpp` on `api_expansion` covers the whole lifecycle, so most of Part A below
is background for *why* it does what it does rather than work you still have to
do. Available now:

| Surface | Notes |
|---|---|
| `SRInterfaceDX9` / `DX11` / `DX12` / `OGL` | `CreateSRInterfaceXxx(...)` → `SetInputTexture` → `Weave()` → `Delete()` |
| Correct init ordering | `SRContext::create` → `CreateXxxWeaver` → `initialize()`, the A.2 gotcha, handled |
| `SRContext::create` / `deleteSRContext` | The documented allocation pair — never `new`/`delete` (the object lives in the SR DLL) |
| Exception containment | `ServerNotAvailableException` and friends become `E_NOINTERFACE`; nothing escapes into your engine |
| Delay-load probing | Core **and** the backend weaver DLL, per backend, before any SDK call (A.3.2) |
| `SetShaderSRGBConversion` / `SetLatencyInFrames` / `SetLatency` / `EnableLateLatching` | On every interface |
| `GetPredictedEyePositions` | The weaver's own prediction — no tracker, no listener, no opt-in |
| `SetCrosstalk(mode, strength, contrast)` | The weaver's own anti-crosstalk, on every interface — see A.11.1. Returns `false` when there is no weaver yet or the runtime doesn't offer the settings interface |
| `SRQueryWeaverSettings` / `SRReadCrosstalkDefaults` / `SRApplyCrosstalk` | The same controls for code that creates its SDK weaver **directly** (no `SRInterfaceXxx`). Header-only, in a section of `SR.hpp` compiled only where the SDK headers are on the include path (`__has_include`), so wrapper-only consumers still need no SR SDK. `SRLib::SDK` exports SR-lib's root for this |
| `SREnableLensHint` / `SRDisableLensHint` / `SRIsLensHintEnabled` | Switchable-lens panels — see A.10. Safe to call every frame: an unchanged preference never reaches the SDK, and a fixed-lens panel costs one failed create, not one per frame. Returns `S_OK` = changed, `S_FALSE` = already there, `E_NOINTERFACE` = no context / no lens |
| `SRRequestTracking` + `SRGetHead` / `SRGetHeadPose` / `SRGetEyePair` | Polled head/eye tracking; the SDK's listener+stream+mutex boilerplate is done for you |

Two things to know before reaching for them:

- **Tracking must be requested before you create an interface.** SR senses have
  to be registered before `SRContext::initialize()`, and `CreateSRInterfaceXxx`
  calls that at the end of its work. So `SRRequestTracking(SR_TRACK_HEAD)` comes
  *first*; calling it afterwards returns `E_NOT_VALID_STATE` rather than
  silently doing nothing. The lens hint has no such constraint — it's created
  lazily on first use.
- **The lens API only sees a context SR-lib made.** It resolves through the
  `SRContext` that `CreateSRInterfaceXxx` creates, so if you drive the SDK
  weavers yourself and own your context, every lens call returns
  `E_NOINTERFACE` — the API is there but unreachable. Going through
  `SRInterfaceXxx` is the fix, and is what you want anyway (A.0, B.3).
- **Maturity is uneven.** The DX12 path, the CMake package, the context
  lifecycle, the DLL probing and the **switchable lens hint** are exercised on
  real hardware (rt64-3D / Goemon64Recomp3D; lens verified 2026-08-10 on a
  switchable-lens panel via UU3D). The DX11 path is wired and building against
  a real integration (UU3D) but its weave is **not yet confirmed on a panel**. The
  **anti-crosstalk raw-SDK helpers** are exercised on real hardware ([VRto3D](https://github.com/oneup03/VRto3D)'s
  raw `IDX11Weaver1`, 2026-10-11), and so are the `SRInterfaceXxx::SetCrosstalk`
  wrappers: `SRInterfaceDX12` in [dusklight3D](https://github.com/oneup03/dusklight3D) (2026-10-19, once its sRGB flag was
  fixed; see pitfall #16) and [UU3D](https://github.com/oneup03/UU3D)'s DX11/DX12 compositors. All tracking,
  the DX9/OGL knobs and the 32-bit build compile and link but have **not** been
  run. Treat those as untested code you're the first to exercise.

### The SDK is nested — relevant only if you bypass the package

SR-lib holds **per-architecture SDK trees**, so `include/` and `lib/cmake/` are
*not* at the checkout root:

```
libs/SR-lib/
├── CMakeLists.txt               ← the package above (api_expansion only)
├── SR.cpp                       ← SR-lib's own wrapper (owns the SRContext lifetime)
├── SR.hpp
└── SR-SDK-<version>/            ← e.g. SR-SDK-1.34.10; globbed, don't hardcode
    ├── x64/                     ← ** this ** is your SDK root
    │   ├── include/
    │   ├── lib/                 ← import libs (.lib) + lib/cmake/
    │   └── third_party/         ← OpenCV, GLog, Leap, asio, rapidxml
    └── x32/                     ← 32-bit tree; SR libs suffixed "32"
```

So the SDK root is `libs/SR-lib/SR-SDK-<version>/x64` — not the submodule root.
`add_subdirectory` works this out for you; the layout is here for when you're
debugging why it didn't.

### ⚠ `find_package` does not work against SR-lib — link the import libs directly

Only relevant if you're building your own CMake glue instead of using the
package (the package already does this correctly).

Using `find_package(simulatedreality)` + `find_package(srOpenGL)` works against a
**full SR SDK installation**, but **fails against the SR-lib submodule**, with a
misleading error:

```
The installation package was faulty
```

Cause: the SDK's shipped CMake configs declare their targets `SHARED IMPORTED`,
and CMake's import check demands the runtime `bin/*.dll` sit alongside each
`.lib`. SR-lib vendors only the **import libraries** — deliberately, since the
runtime DLLs belong to the user's system-wide SR installation and must not be
redistributed (see B.1 *Runtime layout*). The check therefore fails even though
everything needed to link is present.

Skip `find_package` and name the `.lib` files:

```cmake
set(SR_LIBDIR "${LEIASR_SDKROOT}/lib")
set(SR_LIBS
    "${SR_LIBDIR}/simulatedreality.lib"
    "${SR_LIBDIR}/SimulatedRealityCore.lib"
    "${SR_LIBDIR}/SimulatedRealityOpenGL.lib"       # or SimulatedRealityDirectX.lib
    "${SR_LIBDIR}/SimulatedRealityDisplays.lib"     # SwitchableLensHint
    "${SR_LIBDIR}/SimulatedRealityFaceTrackers.lib" # Head/HeadPose/Eye trackers
    "${SR_LIBDIR}/DimencoWeaving.lib"
    "${LEIASR_SDKROOT}/third_party/OpenCV/lib/x64/opencv_world343.lib"
    "${LEIASR_SDKROOT}/third_party/GLog/lib/x64/glog.lib"
)
```

`opencv_world343.lib` is **not optional** even though you never write OpenCV
code: the SR import libs inline `cv::String` destructors, so the linker needs it
to resolve them. Same for `glog`. Omit them and you get unresolved externals
that name symbols you've never heard of.

Nothing is lost by skipping `find_package`: every SR DLL is delay-loaded
(A.3.1), so the import libraries are all the linker ever needed.

## A.1 The SbS input contract — feed the weaver full-SbS (2W × H)

The SR weaver consumes a **single texture** containing both eyes packed side by side (LEFT in `[0, w/2)`, RIGHT in `[w/2, w)`) and a description of its dimensions. The weaver internally samples `uv.x ∈ [0, 0.5]` for the left eye and `uv.x ∈ [0.5, 1]` for the right.

When you call `setInputViewTexture(tex, width, height, format)` (the GL signature; the DX signature is analogous), `width` is the **combined** width of the SbS pair, **not** the per-eye width.

The DVB / HDMI 1.4a terminology has two SbS packing layouts and **you should feed the weaver the "full" one**:

| Layout | Combined dims | Per-eye dims | Quality | Cost |
|---|---|---|---|---|
| Half-SbS (HDMI 1.4a "Side by Side (Half)") | `W × H` (= panel resolution) | `W/2 × H` (horizontal half-rate) | Lower — each eye is sub-sampled; the SR weaver's lenticular sampler must upscale to per-eye panel-column rate | Cheap — fits in one panel-sized backbuffer |
| **Full-SbS (HDMI 1.4a "Side by Side (Full)")** | **`2W × H`** | **`W × H`** (full per-axis rate) | **Higher — each eye is at full per-eye-panel-column resolution; weaver samples 1:1** | Wider intermediate / re-render |

**Recommendation: feed the weaver `2W × H` (full-SbS).** The weaver's lenticular sampler operates at the panel's per-eye column rate; if you give it `W × H` (half-SbS), each eye is only `W/2 × H`, and the weaver bilinear-upscales internally before the lenticular interleave. That bilinear step is a quality loss that propagates into the woven output. With `2W × H` input, each eye is at full per-eye column rate and no internal upscale is needed.

There are two ways to actually produce a `2W × H` source:

1. **Render natively at `2W × H`** (each eye rendered into a `W × H` half of a wider backbuffer). True highest quality — the signal is genuinely `W × H` per eye, no resampling anywhere in the pipeline. Costs 2× the GPU.
2. **Render at `W × H` (half-SbS-packed) and blit-stretch to `2W × H`** in the present path before handing to the weaver. The signal is still effectively `W/2 × H` per eye (the blit is a 2× bilinear upscale), but the weaver receives a `2W × H` input and operates without its own internal upscale. Cheaper — no engine-side render changes required.

Option 2 is what most LeiaSR-aware engines start with — it's a small change isolated to the present path. The RingRacers-3D reference does exactly this: the eye loop produces a `vid.width × vid.height` half-SbS backbuffer (no change from the engine's normal stereo path), and the present-time LeiaSR dispatch performs a `vid.width × vid.height → 2*vid.width × vid.height` blit-stretch before the weave (see B.1).

Option 1 is what you want if "highest quality" matters more than GPU cost. It requires widening the engine's stereo render target and updating eye-viewport math — usually not localized to the present path.

> ⚠ **Don't pass `width/2` thinking "the weaver wants the per-eye width."** The weaver samples exactly `width × height` texels and assumes SbS layout. If you pass `width/2`, it samples only the left half of your texture and produces a corrupt image. Confirmed end-to-end on a real Leia panel via Aurora ([aurora3D `lib/webgpu/leiasr.cpp`](https://github.com/oneup03/aurora3D/blob/Stereo3D-rebased/lib/webgpu/leiasr.cpp), used by dusklight3D) and BanjoRecomp3D (via [rt64-3D](https://github.com/oneup03/rt64-3D)) — both pass the full combined SBS width (`2 * perEyeW`) and weave correctly.
>
> ⚠ **There's a second, separate failure mode that produces the SAME visible symptom — "weave stretched to 2× display width, left half visible."** It's the command-list rasterizer viewport not being reset between a prior compose pass and the weave. See B.2.2 for the diagnosis and fix.
>
> *(Empirical caveat — see B.2.2: the SR DX12 sample and [VRto3D's DX11 LeiaSR presenter](https://github.com/oneup03/VRto3D/blob/main/vrto3d/src/presenter/leiasr_presenter.cpp) both pass **per-eye** width, not the combined width, on those backends. Cross-check what your specific backend's reference code actually ships before settling on either convention.)*
>
> *(Field data, nautymac fork — SR Platform 1.34.10 on a SpatialLabs panel: seven integrations pass the **per-eye** width for a texture that is `2 × eyeW` wide and weave correctly, with depth confirmed by eye — the raw-SDK GL weaver (OGL3D, Azahar-SR), DX11 (geo-11_SR SRWeave, OGL3D upscale mode, VK3D and Azahar-SR via interop), DX12 (DX12_3D, Xenia-3D) and DX9 (TriDef SRWeave). On that runtime the argument evidently does not choose the sampled region; the weaver takes the extent from the texture. So both conventions weave there. Use the documented one (the texture's full width) in new code, but don't blame a corrupt weave on a per-eye value before checking the real texture size and the viewport (B.2.2).)*

> ⚠ **Don't lie about texture dimensions.** Passing `2W` while the texture is actually `W × H` makes the weaver sample past the right edge and produces visible garbage. The dimensions must match the texture's allocated size.

> ⚠ **Don't pass two separate per-eye textures.** The OpenGL weaver API is single-texture SbS only. (Some DX/Vulkan weaver variants accept two textures; the GL `IGLWeaver1` interface as of late 2025 does not.)

## A.2 The SR runtime lifecycle

There is **one global `SR::SRContext` per process**, and **one weaver per `SRContext`**. The lifecycle is:

```
SR::SRContext::create()           ── creates the SR context object
SR::CreateGLWeaver(ctx, hwnd, &w) ── creates the GL weaver (or CreateDX*Weaver, etc.)
ctx->initialize()                 ── ★ AFTER weaver creation. Starts eye tracking.
─── per-frame loop ──────────────
w->setInputViewTexture(tex,...)   ── (re-)bind the SbS source (cache; see A.5)
w->weave()                        ── one weave; writes into current render target
─── shutdown ────────────────────
w->destroy()
SR::SRContext::deleteSRContext(ctx)
```

The **ordering of `ctx->initialize()` is the #1 most-misdiagnosed gotcha**:

- Calling it **after** `CreateGLWeaver` (correct): eye tracking starts, the weaver receives tracked eye coordinates, lenticular interleave works.
- Calling it **before** `CreateGLWeaver`, or **not at all** (wrong): every API call returns success. `weave()` runs. But the lenticular interleave runs against default no-track eye coordinates. The image looks correct as a still — and to anyone debugging without a Leia display in hand — but on the actual Leia panel it shows raw SbS that doesn't respond to head movement. *"The weaver isn't working"* almost always means "the order of these two calls is wrong."

Always log a one-liner on the success path of `Init` so you can verify at runtime that `initialize()` ran and which order it ran in.

### A.2.1 Optional: defer `srk_init` until first weave

The lifecycle above implies "init at process startup, weave per frame." That's the common case and what most hosts ship. There's a reason to split it differently:

**Some SR runtime versions briefly minimize / freeze / re-parent the host window during `SR::SRContext::create()` + `g_ctx->initialize()`.** If your host's startup sequence is "open SDL window, show title screen, eventually start the per-frame loop," running the SR init eagerly at process startup can trash the title screen render — black-frame flash, a single minimize/restore cycle, occasional title-music dropouts on the first second. None of this is permanent; it just looks bad to the user the first time they launch.

The mitigation is to **split the init in two**:

1. **Eager step (at startup):** `LoadLibraryW` / `LoadLibraryA` the shim DLL (if you have one) and resolve its exports. **Don't call any SR API yet.** This proves the shim and its delay-loaded SR deps resolve, without spinning up the SR runtime itself.
2. **Lazy step (on first weave attempt):** when the user actually selects LeiaSR mode and the present path issues its first weave, *then* call `SRContext::create` + `CreateXxxWeaver` + `ctx->initialize`. The window flash happens during gameplay rendering where it's masked by the next frame, instead of during the title screen.

```c
// Eager: just resolve the shim and its exports. No SR API yet.
void R_LeiaSR_LoadShim(void) {
    shim = LoadLibraryA("leiasr_shim.dll");
    if (!shim) return;
    p_init     = (PFN_srk_init)    (void*)GetProcAddress(shim, "srk_init");
    p_weave    = (PFN_srk_weave)   (void*)GetProcAddress(shim, "srk_weave");
    p_shutdown = (PFN_srk_shutdown)(void*)GetProcAddress(shim, "srk_shutdown");
}

// Lazy: defer the SR runtime spin-up to the first weave that needs it.
void R_LeiaSR_Weave(uint32_t tex, int w, int h) {
    if (!p_weave) return;
    if (!sr_init_attempted) {
        sr_init_attempted = true;
        if (p_init && p_init(I_GetWindowHandle())) sr_available = true;
    }
    if (sr_available) p_weave(tex, w, h);
}
```

Use this pattern if you have observed startup-time SDL window misbehavior with eager init. If your host doesn't show that symptom, plain eager init is simpler and fine.

### A.2.2 Make the process per-monitor-DPI-aware BEFORE init (non-negotiable on scaled displays)

The lenticular weave only produces correct autostereo when it lands **1:1 on physical panel pixels**. If the host process is not per-monitor-DPI-aware and the SR panel (or any monitor the window sits on) is set to >100% Windows display scale, the OS maps your window — and therefore the weaver's output — into a **DPI-virtualized sub-region** of the native panel and then stretches it up. The weave no longer aligns to the lens, and the 3D collapses into blur/ghosting. This has bitten several of the reference integrations repeatedly and is *distinct* from the wrong-init-order gotcha (A.2) — the order can be perfect and the weave still breaks purely from DPI virtualization.

**Every SR SDK sample sets `PROCESS_PER_MONITOR_DPI_AWARE` at startup.** Match that. The fix is a process-init action, not an SR API call, and it must run **before the host creates its window or swap chain** (once the window exists, the awareness context is baked in):

```cpp
// As early as possible — DllMain(DLL_PROCESS_ATTACH) for an injected mod, the
// SR init entry, or the app manifest for a native app. Before the first window.
SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);
// fallbacks: SetProcessDpiAwareness(PROCESS_PER_MONITOR_DPI_AWARE) / SetProcessDPIAware()
```

Two practical notes:

- **Re-assert it on the SR init thread.** If SR init runs on its own thread (or you defer it per A.2.1), call `SetThreadDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2)` on that thread before `SRContext::create()` / `initialize()`, and verify with `GetAwarenessFromDpiAwarenessContext(GetThreadDpiAwarenessContext())`. Log per-monitor / system / unaware so you can confirm at runtime that the awareness actually took and injection landed early enough.
- **This is the *prerequisite* for pitfall #11, not a duplicate of it.** Even a correctly PerMonitorV2-aware host still gets its fullscreen window resized with DPI-virtualized coordinates by the SR service ~1 s after init — that residual, race-y one-shot resize needs the re-assert-the-rect fix in pitfall #11. Being DPI-aware is necessary but not sufficient; you need both. (See also `output3d` §3.1.1 — the same physical-pixel requirement underpins the interlaced/checkerboard modes.)

## A.3 Shipping a binary that runs on machines without the SR runtime

This is the section most LeiaSR integrations get wrong, and the symptom is brutal: **on any machine that has never had the LeiaSR installer run, your `.exe` shows a popup at startup and refuses to launch**, looking something like:

> "The code execution cannot proceed because **opencv_world343.dll** was not found. Reinstalling the program may fix this problem."

This pops up **before `main()` runs** — there's no opportunity to catch it. The user sees an unprofessional Windows loader popup and concludes your game is broken.

The fix has two layers; you need both.

### A.3.1 `/DELAYLOAD` every DLL the SR libs pull into your import table

When you link the SR import libraries normally, the linker writes hard imports into your `.exe`'s PE import table for every SR DLL whose functions you reference, *plus their transitive imports that the linker can see*. The OS loader resolves all hard imports at process start — that's where the popup comes from.

`/DELAYLOAD` flips a specific DLL's imports to lazy: the OS loader skips it at startup, and Windows only tries to resolve it on the first call into one of its imported functions. With every SR-touched DLL on the list, the EXE starts cleanly even when none of those DLLs exist on disk.

CMake (MSVC) recipe:

```cmake
if(WIN32 AND HAVE_LEIASR AND MSVC)
    target_link_libraries(MyExe PRIVATE delayimp)
    target_link_options(MyExe PRIVATE
        /DELAYLOAD:simulatedreality.dll
        /DELAYLOAD:SimulatedRealityCore.dll
        /DELAYLOAD:SimulatedRealityCameras.dll
        /DELAYLOAD:SimulatedRealityHandTrackers.dll
        /DELAYLOAD:SimulatedRealityFaceTrackers.dll
        /DELAYLOAD:SimulatedRealityUserModelers.dll
        /DELAYLOAD:SimulatedRealityDisplays.dll
        /DELAYLOAD:SimulatedRealityOpenGL.dll       # OpenGL backend
        /DELAYLOAD:SimulatedRealityDirectX.dll      # DirectX backend (add if linked)
        /DELAYLOAD:DimencoWeaving.dll
        /DELAYLOAD:opencv_world343.dll              # ← THE KILLER
    )
endif()
```

> **`/DELAYLOAD:opencv_world343.dll` is critical and easy to miss.** The SR import .libs inline a `cv::String::deallocate` call (in destructors), which pulls a single OpenCV symbol into your EXE's import table. You don't write OpenCV code; you don't `target_link_libraries(opencv)`; OpenCV doesn't appear anywhere in your project — but the linker still emits real `opencv_world343.dll` imports because of the inline. This is the DLL that the loader popup almost always names.

`/DELAYLOAD` is a no-op for DLLs not actually imported, so erring on the side of "list more" is harmless. If a future SDK version pulls in a new DLL, your test on a clean machine will show a popup naming the new DLL — add it to the list.

**Verify with `dumpbin`** before shipping:

```powershell
dumpbin /imports MyExe.exe | findstr ".dll"
```

Output is in two sections: "Section contains the following imports" (top-level / hard imports) and "Section contains the following delay load imports". **Every non-system, non-CRT DLL must appear ONLY in the delay-load section.** If `opencv_world343.dll` (or any SR DLL) is in the top section, the EXE will fail to launch on a clean machine — re-run the build after adding the missing `/DELAYLOAD:` line.

Sample acceptable output:

```
Section contains the following imports:
    dbghelp.dll                ← system
    KERNEL32.dll               ← system
    USER32.dll                 ← system
    [... other Windows DLLs ...]

Section contains the following delay load imports:
    SimulatedRealityOpenGL.dll      ← good
    SimulatedRealityCore.dll        ← good
    opencv_world343.dll             ← good
```

### A.3.2 `LoadLibraryW` preflight, because SEH from missing DLLs isn't catchable

`/DELAYLOAD` alone is not enough. If the DLL still isn't there when the first imported function gets called, the **delay-load helper raises an SEH exception** (`VcppException`, code `0xC06D007E`). C++ `try`/`catch (...)` **does not catch SEH** unless the whole TU is compiled with `/EHa`, which changes the ABI for every function in it. You really don't want `/EHa` ABI-wide just for one error path.

The standard fix is to **probe up-front with `LoadLibraryW`** and bail before any SR call is attempted. `LoadLibraryW` is a normal Win32 function — it returns `NULL` on failure instead of raising SEH:

```cpp
static bool sr_runtime_available()
{
    // One probe is enough — the SR DLLs install as a set, and DimencoWeaving
    // has opencv_world343 as a transitive dependency, so a successful
    // LoadLibraryW here proves BOTH the SR runtime and opencv resolve.
    HMODULE h = LoadLibraryW(L"DimencoWeaving.dll");
    if (!h) return false;
    FreeLibrary(h);   // optional — see "Don't FreeLibrary?" note below
    return true;
}
```

Gate every SR API call behind this:

```cpp
bool R_LeiaSR_Init(void* hwnd)
{
    if (!sr_runtime_available()) {
        log_notice("LeiaSR runtime not installed. Falling back to SbS.");
        return false;
    }
    // ...safe to call SR API from here on...
}
```

**Why `DimencoWeaving.dll` specifically?** It's the deepest piece of the SR runtime chain (the weaver implementation) and has `opencv_world343.dll` as a transitive dep. Loading it forces Windows to resolve everything else. Any of the SR DLLs would work — pick one — but the convention is to probe the deepest one.

**Probe the backend weaver DLL, not just the core runtime.** A single probe of `SimulatedRealityCore.dll` is *not* enough: a machine can have the core runtime present while the backend weaver DLL is missing, which clears the guard and then raises SEH inside `CreateXxxWeaver` — exactly what the preflight exists to prevent. Probe the DLL your backend's factory actually lives in: `SimulatedRealityDirectX.dll` for DX9/11/12, `SimulatedRealityOpenGL.dll` for GL (32-bit builds suffix all of these with `32`).

That probe is also the strongest single check available. Verified with `dumpbin /dependents` against an installed LeiaSR Platform runtime, `SimulatedRealityDirectX.dll` statically imports:

```
d3d12.dll  SimulatedRealityFaceTrackers.dll  SimulatedRealityDisplays.dll
DimencoWeaving.dll  SimulatedRealityCore.dll  opencv_world343.dll
```

— so a successful `LoadLibraryW` on it resolves the entire chain including OpenCV. Probing core as well costs one refcount bump and doesn't depend on that dependency set holding for every SDK build, so do both.

`SRLib::SR` (A.0) already does this per backend, and probes on every create rather than once globally, since each backend brings a different DLL. If you're using the wrapper you get this for free — and a host that was carrying its own probe purely to cover the backend DLL can drop it.

> **Don't `FreeLibrary`?** You'll see both conventions in the wild. Calling `FreeLibrary` keeps the probe side-effect-free (the SR runtime isn't loaded until you actually start using it). Leaving it loaded saves a future `LoadLibraryW`/`FreeLibrary` cycle when the SR DLLs actually load. Either is fine. The RingRacers-3D implementation calls `FreeLibrary`; the dx12 reference implementation doesn't. Pick one.

> **`/EHa` is acceptable when the entire TU is the SR shim.** The skill discourages compiling with `/EHa` "ABI-wide" because it expands the meaning of `catch (...)` for *every function in the translation unit*. In a host engine TU that's a wide blast radius. In the **C-ABI shim DLL pattern** (B.1 below), the entire shim TU exists for SR integration only — there's no non-SR code in scope — so `/EHa` is a precise tool, not a wide one. Combining `/EHa` *with* the `LoadLibraryW` preflight is belt-and-suspenders: the preflight fails fast on the no-runtime path (cheaper than throwing an SEH out of `SR::SRContext::create()`), and `/EHa` catches any SEH that slips past (e.g. a partial runtime install where `DimencoWeaving.dll` resolves but a sibling doesn't). Both layers are cheap and complementary.

### A.3.3 Layered exception handling — both layers, both reasons

After the preflight passes, **C++ exceptions from the SR API are still possible** (and common):

- `SR::SRContext::create()` can throw if no SR display is connected.
- `SR::CreateGLWeaver()` can throw if the GL context isn't compatible.
- `weaver->weave()` can throw if the SR service crashes mid-session, or if the user unplugs the SR display.

These are normal C++ exceptions — they **do** get caught by `try { ... } catch (const std::exception&) { ... } catch (...) { ... }`. So you want the full pattern:

```
LoadLibraryW preflight  ──── guards against SEH from missing delay-loaded DLLs
try/catch around SR API ──── guards against std::exception from missing hardware
```

Both layers are necessary. Without the preflight, a missing DLL crashes the process before main(). Without the try/catch, a display unplugged mid-session crashes the process.

The reference implementation in A.6 below shows both layers wired together.

## A.4 Upscaling from a low render resolution to the SR panel's native resolution

**You do not have to render at the SR panel's native resolution.** The weaver handles upscaling internally.

How it works: `setInputViewTexture(tex, w, h, fmt)` describes the input texture's *actual* dimensions. `weave()` writes into the **currently bound render target at the current viewport** — typically the SR display's full-window viewport. The weaver's internal sampling does whatever filtering is needed to map the input `w × h` SbS image onto the output viewport's pixels.

Concrete: on a 4K SR panel, you can render at 1080p (1920×1080 SbS, each eye effectively 960×1080), hand the 1920×1080 texture to the weaver, and the weaver produces a 4K woven output on the panel. The lenticular interleave still happens at output resolution — the upscale is only applied to the pre-weave SbS source.

Recommendations:

- **Match render res to display res** for the sharpest output. The weaver doesn't filter when no scaling is needed.
- **Half-res per axis** (e.g. render 1920×1080 SbS for a 3840×2160 panel) is a common choice on weaker GPUs. The bilinear upscale inside the weaver is smooth enough that this works well; the lenticular pattern is sharp because it's evaluated at panel resolution.
- **Don't render at lower-than-display-resolution in only one axis** — anisotropic downsample is uglier than isotropic.

### Filtering pitfall

Set the source texture's min/mag filter to **`LINEAR`** before handing it to the weaver. The default for a freshly-created FBO in many engines is `NEAREST`, which makes the upscale blocky. One `glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR)` (and matching `_MAG_FILTER`) suffices for the OpenGL path. RHI-based engines typically expose `update_texture_settings` / equivalent.

If your engine renders into an FBO that's used both for in-game post-processing (often `NEAREST` for pixel-accurate effects) and for the weaver hand-off, make this `LINEAR` switch in the present path, **not at FBO creation time**. The post-processing wants `NEAREST` still.

### When the host renders SbS to its full backbuffer, the weaver sees it as-is

The simplest LeiaSR integration is "render SbS to backbuffer at render resolution; hand backbuffer texture to weaver; weaver writes to default framebuffer at window resolution." No separate "stretch + recapture" pass needed — the weaver's own sampling is the stretch.

Some older guides (written for Doom-lineage engines whose legacy present path is `OglSdlFinishUpdate`) recommend a `capture → stretch → recapture → weave` pattern. **That was for engines whose backbuffer *is* the default framebuffer**, where you had to capture, scale, and re-capture to get a separate intermediate texture for the weaver. **If your engine has the rendered SbS image in a standalone FBO color texture, you skip the stretch + recapture — pass the FBO texture directly to the weaver.** The RingRacers-3D HWR2/RHI architecture does exactly this.

## A.5 Caching `setInputViewTexture`

`setInputViewTexture` is not free. It tells the weaver to re-bind its sampling source, which on some SR versions reinitializes internal state. Calling it every frame works, but adds latency on lower-end GPUs.

Cache the last (tex_id, w, h, format) tuple in the wrapper and skip the call when nothing changed:

```cpp
void R_LeiaSR_Weave(uint32_t tex_id, int w, int h, uint32_t fmt) {
    if (tex_id != cached_tex || w != cached_w || h != cached_h || fmt != cached_fmt) {
        weaver->setInputViewTexture(tex_id, w, h, fmt);
        cached_tex = tex_id; cached_w = w; cached_h = h; cached_fmt = fmt;
    }
    weaver->weave();
}
```

If your engine never re-creates the SbS texture, this means `setInputViewTexture` is called exactly once and `weave()` runs alone every frame thereafter.

## A.6 Reference implementation (annotated)

> ⚠ **Read A.0 first — SR-lib's `SR.cpp` is a shipped, hardware-tested version of
> everything below.** If you linked `SRLib::SR` you already have the lifecycle,
> the init ordering, the exception containment and the DLL probing, and writing
> this by hand means maintaining a second copy of it. This section stays because
> it is the annotated explanation of *why* the wrapper is shaped this way, and
> because a C-ABI host (the MinGW shim in B.1) still needs a flat surface of its
> own on top. Reach for it when you need a C surface or a host-specific shape —
> not as the default starting point.

This is a complete, copy-pasteable wrapper. It exposes a plain C surface (`R_LeiaSR_*`) so the host engine — typically a mix of C and C++ — can call it without dragging the SR headers everywhere. The `#ifdef HAVE_LEIASR` gate plus the stub bodies in the `#else` branch mean the host engine compiles cleanly on platforms / configurations without SR — no `#ifdef` chains leak into call sites.

### `stereo_leiasr.hpp`

```cpp
#ifndef STEREO_LEIASR_HPP_
#define STEREO_LEIASR_HPP_

#include <cstdint>

#ifdef __cplusplus
extern "C" {
#endif

// Lazy / idempotent. Returns true on success, false if the SR runtime is
// unavailable or no SR display is connected.
//   hwnd: opaque OS window handle (HWND on Windows). Void* keeps the header
//         free of <windows.h> for callers that don't need it.
bool R_LeiaSR_Init(void* hwnd);

// True after a successful R_LeiaSR_Init. Used by the present-time dispatch
// to choose between the LeiaSR path and the SbS fallback.
bool R_LeiaSR_Available(void);

// Teardown. Safe to call when uninitialized.
void R_LeiaSR_Shutdown(void);

// Per-frame weave. Call with the SbS source texture's GL name and the
// texture's actual dimensions and pixel format. Writes into whatever GL
// framebuffer / viewport is currently bound (typically the SDL window).
//   tex_id:  GL texture name (output of glGenTextures, glGetIntegerv, etc.)
//   width:   texture width in pixels (= combined L+R width)
//   height:  texture height in pixels
//   format:  GL_RGB / GL_RGBA / etc. — must match the texture's storage
//            format. 0x1907 = GL_RGB.
void R_LeiaSR_Weave(uint32_t tex_id, int width, int height, uint32_t format);

#ifdef __cplusplus
}
#endif

#endif
```

### `stereo_leiasr.cpp`

```cpp
#include "stereo_leiasr.hpp"

#ifdef HAVE_LEIASR

#include <exception>
#include <mutex>

#define WIN32_LEAN_AND_MEAN
#include <windows.h>

#include <sr/management/srcontext.h>
#include <sr/weaver/glweaver.h>

#include "log.h"   // host engine's logging — replace as needed

namespace {

// A.3.2 preflight — probe whether the SR runtime DLLs resolve. /DELAYLOAD
// defers the SR DLL load to first imported call; if the DLL isn't on disk
// that first call raises an SEH the C++ try/catch can't catch. LoadLibraryW
// returns NULL instead, so we can gate cleanly.
//
// One probe is enough — the SR DLLs ship as a set, and DimencoWeaving has
// opencv_world343 as a transitive dependency. A successful LoadLibraryW here
// proves the whole SR + OpenCV chain resolves.
bool sr_runtime_available()
{
    HMODULE h = LoadLibraryW(L"DimencoWeaving.dll");
    if (!h) return false;
    FreeLibrary(h);
    return true;
}

SR::SRContext*    g_sr_context     = nullptr;
SR::IGLWeaver1*   g_sr_weaver      = nullptr;
bool              g_init_attempted = false;
bool              g_init_succeeded = false;
std::mutex        g_init_mutex;

// A.5 cache
uint32_t g_last_tex_id = 0;
int      g_last_w = 0, g_last_h = 0;
uint32_t g_last_fmt = 0;

void destroy_weaver_locked()
{
    if (g_sr_weaver) {
        try { g_sr_weaver->destroy(); } catch (...) {}
        g_sr_weaver = nullptr;
    }
    if (g_sr_context) {
        try { SR::SRContext::deleteSRContext(g_sr_context); } catch (...) {}
        g_sr_context = nullptr;
    }
    g_last_tex_id = 0;
    g_last_w = g_last_h = 0;
    g_last_fmt = 0;
}

} // namespace

extern "C" bool R_LeiaSR_Init(void* hwnd)
{
    std::lock_guard<std::mutex> lock(g_init_mutex);
    if (g_init_succeeded) return true;
    if (g_init_attempted) return false;   // tried once, failed
    g_init_attempted = true;

    if (!sr_runtime_available()) {
        log_notice("LeiaSR runtime not installed. Falling back to SbS.");
        return false;
    }

    try {
        g_sr_context = SR::SRContext::create();
        if (!g_sr_context)
            throw std::runtime_error("SRContext::create returned null");

        HWND hwnd_native = reinterpret_cast<HWND>(hwnd);
        WeaverErrorCode rc =
            SR::CreateGLWeaver(*g_sr_context, hwnd_native, &g_sr_weaver);
        if (rc != WeaverErrorCode::WeaverSuccess || g_sr_weaver == nullptr)
            throw std::runtime_error("SR::CreateGLWeaver failed");

        // ★ AFTER CreateGLWeaver. Starts eye tracking. Without this, weave()
        // silently emits raw SbS that doesn't respond to head movement.
        g_sr_context->initialize();
    }
    catch (const std::exception& e) {
        log_warning("LeiaSR init failed: %s. Falling back to SbS.", e.what());
        destroy_weaver_locked();
        return false;
    }
    catch (...) {
        log_warning("LeiaSR init failed (unknown exception). Falling back.");
        destroy_weaver_locked();
        return false;
    }

    g_init_succeeded = true;
    log_info("LeiaSR initialized successfully.");
    return true;
}

extern "C" bool R_LeiaSR_Available(void)
{
    return g_init_succeeded && g_sr_weaver != nullptr;
}

extern "C" void R_LeiaSR_Shutdown(void)
{
    std::lock_guard<std::mutex> lock(g_init_mutex);
    destroy_weaver_locked();
    g_init_attempted = false;
    g_init_succeeded = false;
}

extern "C" void R_LeiaSR_Weave(uint32_t tex_id, int width, int height,
                                uint32_t format)
{
    if (!g_init_succeeded || g_sr_weaver == nullptr) return;

    try {
        if (tex_id != g_last_tex_id || width != g_last_w
            || height != g_last_h || format != g_last_fmt)
        {
            g_sr_weaver->setInputViewTexture(
                static_cast<GLuint>(tex_id), width, height,
                static_cast<GLenum>(format));
            g_last_tex_id = tex_id;
            g_last_w = width; g_last_h = height;
            g_last_fmt = format;
        }
        g_sr_weaver->weave();
    }
    catch (const std::exception& e) {
        log_warning("LeiaSR weave failed: %s. Disabling for this session.", e.what());
        std::lock_guard<std::mutex> lock(g_init_mutex);
        destroy_weaver_locked();
        g_init_succeeded = false;
    }
    catch (...) {
        log_warning("LeiaSR weave failed (unknown). Disabling.");
        std::lock_guard<std::mutex> lock(g_init_mutex);
        destroy_weaver_locked();
        g_init_succeeded = false;
    }
}

#else  // !HAVE_LEIASR — stubs so callers compile without SR support

extern "C" bool R_LeiaSR_Init(void*)                          { return false; }
extern "C" bool R_LeiaSR_Available(void)                      { return false; }
extern "C" void R_LeiaSR_Shutdown(void)                       { }
extern "C" void R_LeiaSR_Weave(uint32_t,int,int,uint32_t)     { }

#endif
```

### `CMakeLists.txt` excerpt

> ⚠ **If you vendored SR-lib per A.0, you don't need any of this** — three lines
> of `add_subdirectory` + `SRLib::SR` + `srlib_apply_delayload` do all of it, and
> stay correct as the SDK changes. This longhand form is kept for hosts building
> against a **full SR SDK installation** (where `find_package` does work), and so
> the `/DELAYLOAD` list below is visible in one place. Against the SR-lib
> submodule `find_package` fails with "the installation package was faulty" —
> SR-lib ships import libs without the runtime DLLs the SDK's `SHARED IMPORTED`
> targets check for.

```cmake
option(MYENGINE_ENABLE_LEIASR "Enable LeiaSR autostereoscopic stereo output (Windows only)" ON)

if(MYENGINE_ENABLE_LEIASR AND WIN32)
    # Full SDK installation only. For the SR-lib submodule, see A.0.
    list(APPEND CMAKE_PREFIX_PATH "${LEIASR_SDK_INSTALL}/lib/cmake")
    find_package(simulatedreality REQUIRED)
    find_package(srOpenGL        REQUIRED)   # or srDirectX for the DX backend

    target_link_libraries(MyExe PRIVATE simulatedreality srOpenGL::srOpenGL)
    target_compile_definitions(MyExe PRIVATE HAVE_LEIASR)

    if(MSVC)
        # A.3.1 — delay-load every SR-touched DLL so the EXE launches on
        # machines without the SR runtime installed.
        target_link_libraries(MyExe PRIVATE delayimp)
        target_link_options(MyExe PRIVATE
            /DELAYLOAD:simulatedreality.dll
            /DELAYLOAD:SimulatedRealityCore.dll
            /DELAYLOAD:SimulatedRealityCameras.dll
            /DELAYLOAD:SimulatedRealityHandTrackers.dll
            /DELAYLOAD:SimulatedRealityFaceTrackers.dll
            /DELAYLOAD:SimulatedRealityUserModelers.dll
            /DELAYLOAD:SimulatedRealityDisplays.dll
            /DELAYLOAD:SimulatedRealityOpenGL.dll
            /DELAYLOAD:DimencoWeaving.dll
            /DELAYLOAD:opencv_world343.dll      # THE KILLER — don't omit
        )
    endif()
endif()
```

## A.7 Window-handle plumbing

The weaver needs the OS window handle (Windows: `HWND`) of the render surface so it can install eye-tracking input hooks.

**SDL2:**

```c
void* I_GetWindowHandle(SDL_Window* win) {
#ifdef _WIN32
    SDL_SysWMinfo info;
    SDL_VERSION(&info.version);
    if (SDL_GetWindowWMInfo(win, &info) && info.subsystem == SDL_SYSWM_WINDOWS)
        return (void*)info.info.win.window;
#endif
    return NULL;
}
```

**SDL3:** `SDL_GetPointerProperty(SDL_GetWindowProperties(win), SDL_PROP_WINDOW_WIN32_HWND_POINTER, NULL)`.

**GLFW:** `glfwGetWin32Window(window)` returns the HWND directly. (Requires `#define GLFW_EXPOSE_NATIVE_WIN32` and `<GLFW/glfw3native.h>`.)

**Raw Win32:** you already have the HWND.

Pass as `void*` to the wrapper to keep the header free of `<windows.h>`.

## A.8 Present-path dispatch (engine-side glue)

The host engine's stereo-mode dispatch should branch on the user-selected mode. The LeiaSR branch is conditional on **both** "user picked LeiaSR" *and* "init succeeded":

```cpp
bool leiasr_dispatch =
    (cv_stereomode.value == STEREO_LEIASR)
    && R_LeiaSR_Init(I_GetWindowHandle())   // lazy + idempotent
    && R_LeiaSR_Available();                // post-init check

if (leiasr_dispatch) {
    // Bind default framebuffer + full-window viewport, then weave.
    bind_default_framebuffer_and_full_window_viewport();
    set_texture_filter_linear(stereo_backbuffer);  // A.4 filtering pitfall
    R_LeiaSR_Weave(get_native_texture(stereo_backbuffer),
                   backbuffer_width, backbuffer_height,
                   GL_RGB /* 0x1907 */);
} else if (cv_stereomode.value == STEREO_LEIASR) {
    // User picked LeiaSR but runtime/hardware unavailable — SbS fallback.
    present_sbs_fallback();
} else {
    // Other stereo modes / mono.
    present_normal();
}
```

`R_LeiaSR_Init` is idempotent — calling it every frame after the first success is free.

### Don't move the viewport between framebuffer bind and `weave()`

A specific gotcha: if your engine has a "viewport from rendered backbuffer dimensions" helper (the SRB2 lineage's `GClipRect` is one example), **do not invoke it between binding the default framebuffer and calling `weave()`**. Such helpers compute viewport math against the *rendered* backbuffer size, which may be smaller than the window. The weaver writes its lenticular output at the current viewport — if you've shrunk the viewport to the rendered backbuffer size, the woven image ends up sub-window-sized with negative coordinates.

After binding the default framebuffer, the viewport should be the full SDL window. Leave it alone before `weave()`.

## A.9 Common pitfalls (all backends)

1. **Missing `/DELAYLOAD:opencv_world343.dll`.** The most common shipping bug. The SR import libs inline `cv::String` destructors. `dumpbin /imports MyExe.exe | findstr opencv` reveals it. Adding the line is the entire fix.

2. **`ctx->initialize()` before `CreateGLWeaver`, or omitted.** Silently produces an un-woven image. The image looks fine to non-Leia debugging but no 3D ever appears on the actual panel. **Always**: `SRContext::create()` → `CreateXxxWeaver()` → `ctx->initialize()`.

3. **C++ `try`/`catch (...)` around the first SR call without a `LoadLibraryW` preflight.** Doesn't catch the SEH from a missing delay-loaded DLL. You need both layers.

4. **Wrong dimensions to `setInputViewTexture`.** The weaver samples exactly `width × height` texels. Lying produces garbage along the right/bottom edges. Pass the texture's actual dimensions; pass the *combined* SbS width (not per-eye).

5. **Re-initializing every frame.** `R_LeiaSR_Init` must be idempotent. The `g_init_succeeded` flag handles this in the reference implementation.

6. **`NEAREST`-filtered source texture during weaver upscale.** Blocky upscale artifact. Set `LINEAR` on the source texture before the weave call.

7. **Calling Init/weave from the wrong thread.** The weaver hooks eye tracking via the HWND and expects to be on the thread that owns the render context. Cross-thread calls leave the weaver in a partially-initialized state.

8. **Forgetting `R_LeiaSR_Shutdown` on exit.** The SR runtime is a process-global singleton; leaking it across game restarts can leave the SR service in a degraded state.

9. **Linking against the wrong bitness.** SR ships separate x86 / x64 import libs. Mismatch produces "unresolved external" at link time or a runtime crash when the calling conventions disagree.

10. **Display unplug mid-session.** `weave()` throws when the SR display vanishes. The reference implementation catches it, logs a warning, sets `g_init_succeeded = false`, and falls back to SbS on subsequent frames. Without this guard, the game crashes when the user unplugs the panel.

11. **The SR service resizes your fullscreen window with DPI-virtualized coordinates.** (Prerequisite: the host must already be per-monitor-DPI-aware — see A.2.2; this pitfall is the residual issue that remains *even when it is*.) Shortly after SR init (~1 s, racy — some launches escape it), the SR runtime/service repositions the host window onto its display via `SetWindowPos` from a **DPI-unaware context**. On a monitor at >100% Windows display scale, the OS multiplies that request by the scale factor: a PerMonitorV2-aware host's 3840×2160 fullscreen window becomes physically **5760×3240 at 150%** — larger than the screen. Only the top-left panel-sized region is visible, so the image looks *zoomed/cropped by exactly the DPI factor* and the weave no longer lands 1:1 on panel pixels (3D breaks). Diagnose by polling `GetClientRect` from a `SetThreadDpiAwarenessContext(PER_MONITOR_AWARE_V2)` thread: the client jumps from panel size to panel×scale about a second after launch. Fix: in the host's resize handling, detect a fullscreen window that exceeds its display bounds and re-assert the display rect via `SetWindowPos` from the (DPI-aware) host process. The SR service accepts the correction — it's a one-shot resize, no fight loop. Reference: `fix_dpi_unaware_external_resize()` in [aurora3D `lib/window.cpp`](https://github.com/oneup03/aurora3D/blob/Stereo3D-rebased/lib/window.cpp) (the `Stereo3D-rebased` branch; dusklight3D consumes it as `extern/aurora`).

12. **Losing `NOMINMAX` when you stop including SDK headers.** The SDK's `sr/world/display/display.h` defines `NOMINMAX` before it reaches `windows.h`, so any TU including `sr/management/srcontext.h` gets that for free. `SR.hpp` deliberately doesn't include SDK headers (it pulls `d3d9/11/12.h` only), so a TU that migrates from the SDK to the facade suddenly gets windows.h's `min`/`max` **macros** and every `std::min` / `std::max` in the file fails with `C2589: '(': illegal token on right side of '::'`. Define `NOMINMAX` yourself, before the first include, rather than inheriting it by accident from an unrelated header.

13. **Putting SR SDK headers in your own headers.** `SRContext` transitively includes the SDK's bundled OpenCV headers, which fail to compile in unrelated TUs (`opencv2/flann/kdtree_single_index.h` → ~100 × `C3861: identifier not found`). Keep SDK includes in `.cpp` files and forward-declare in headers — or just use `SR.hpp`, which is SDK-header-free precisely so it *can* live in your headers.

14. **A wedged SR runtime looks exactly like an integration bug.** The service can reach a state where output is flat mono while **every** API reports health: create/weave succeed and never throw, geometry is pixel-exact, eye tracking returns live moving positions, `GetBehaviorWhenNotTracking()` is `Default` (weave as usual), and the lens hint is accepted. Observed 2026-08-10 across two games; relaunching, re-creating the weaver and toggling output modes all failed, and **only a Windows restart cleared it**. If your integration passes A.12 and still shows mono, restart Windows before you start bisecting code. Two theories worth *not* re-deriving, both falsified in that session: a backbuffer smaller than the panel (the per-eye render/eye-texture size is not the swap-chain size — the compose upscales, so in-game resolution changes are not a LeiaSR problem), and a host that lies about DPI to the SR runtime (faking the DPI *value* doesn't move window/monitor rects, which stay physical in a per-monitor-aware process).

15. **DPI awareness declared too late (SDL hosts especially).** Process DPI awareness is **one-shot** — the first declaration wins and later calls silently fail. SDL declares it during `SDL_VideoInit`/`SDL_Init(SDL_INIT_VIDEO)`, so a host that calls `SetProcessDpiAwarenessContext` *after* that gets a no-op and quietly runs at whatever SDL chose. Declare it **before** SDL touches video, also set `SDL_SetHint("SDL_WINDOWS_DPI_AWARENESS", "permonitorv2")` so SDL doesn't claim something weaker first, and **read it back** (`GetAwarenessFromDpiAwarenessContext(GetThreadDpiAwarenessContext())`) and log per-monitor / system / unaware — otherwise you cannot tell whether A.2.2 actually took effect.

16. **`setShaderSRGBConversion(false, false)` on display-encoded content.** This is the default trap for any engine whose swap chain is plain `*_UNORM`, which describes almost all of them, because "UNORM" reads as "linear". Colours look completely normal either way, since the read and write conversions mirror each other. Ghosting is just worse than it should be, and the anti-crosstalk Strength control (A.11.1) seems to do nothing, because the weaver's linear-light cancellation is being applied to encoded values. Use `(true, true)` unless your pipeline really holds linear values (B.2 gamma note). Flip your compose shader's ghost-reduction space at the same time (A.11.2).

17. **A 32-bit game hangs or crashes the moment the weaver is created.** The log ends after the SR context was created and never reports the weaver. The 32-bit runtime (`DimencoWeaving32.dll`) probes for Microsoft Virtual PC with an illegal instruction it expects to catch with its own SEH frame, and a vectored exception handler in the host gets it first. Mono's does exactly that, so expect it in any 32-bit Unity game (found in Absolute Drift). Fix and diagnosis: A.13.1. x64 is not affected.

18. **The desktop stays lensed after the game died.** A process killed with the lens hint on leaves the panel lensed. Neither a new client's `disable()` nor restarting `SRSession.exe` clears the stale hint; restarting the `SR Service` Windows service does. See A.10's last paragraph.

19. **Polling the lens state per frame.** `SwitchableLensHint::isEnabled()` is an IPC round trip to the SR service, and it measured about **45 ms per call** (SR Platform 1.34). Read it at a mode switch or in a diagnostic, never in the present path. Track what you asked for instead (A.10).

20. **The exit hangs and the tracking camera stays on.** `destroy()` on a DX11 weaver waits for the weaver's processing threads, and those talk to the render window's thread. In an emulator that stops emulation on its emulation thread while the GUI thread waits for it, destroying the weaver there deadlocked the exit: the process lost its window, kept running, and kept the SR context and camera alive (SR log ends at "Lens disable"). Retire weavers to a cleanup thread (A.13.5).

21. **Black weave from a 10-bit input.** With the DX12 weaver, an `R10G10B10A2_UNORM` input (Xenia's guest output) wove to black, also through a stable copy. The same picture composed into a `B8G8R8A8_UNORM` texture first wove correctly. Hand the weaver an 8-bit-per-channel input unless you have verified another format on your runtime.

22. **The input copy of an sRGB back buffer.** If the copy is a typed `*_UNORM_SRGB` texture, `CreateShaderResourceView` with any other format fails (`E_INVALIDARG` every frame, so weaving never starts). If the input view and the output RTV differ in sRGB-ness, the gamma is encoded twice and the woven picture is too bright. Create the copy **typeless**, and give the input SRV and the output RTV the same format (A.13.3). This is a separate trap from #16, which is about the weaver's own conversion flags.

23. **Exclusive fullscreen at a resolution other than the panel's.** The game's back buffer is scaled to the panel after weaving, so the lenticular pattern no longer lines up with the lens and the 3D collapses. A.12 #8 checks the sizes. A game that offers no native-resolution mode needs borderless at native resolution, or an upscale path (A.13.4) that weaves at the panel's resolution. Log a warning when back buffer and window client differ; a user can fix that themselves.

## A.10 The switchable lens is global state — hand it back

Some SR panels have a switchable lenticular layer: lens off is an ordinary
sharp 2D monitor, lens on is autostereoscopic. `SREnableLensHint()` /
`SRDisableLensHint()` express a *preference*, and the SR service ORs the
preferences of every connected application — so **the lens stays on while any
app wants it on**, including yours after you've stopped weaving.

That makes releasing it a correctness requirement, not politeness. An
integration that only ever enables leaves the panel lensed over every other 3D
output mode you offer, over other applications, and over the desktop behind
your window.

Three rules:

- **Drive it from whether you actually wove, not from which mode is selected.**
  A frame that selected LeiaSR but fell back to SbS (weaver creation failed,
  runtime absent, mid-session loss) must release too. `set_lens(weave_succeeded)`
  once per present is the whole policy, and it's cheap — SR-lib skips unchanged
  preferences, so it costs a bool compare on the frames where nothing moved.
  (SR-lib's own header suggests following window focus/visibility instead; either
  is defensible. Weave state is the stricter signal and needs no window hooks.)
- **Release BEFORE you destroy the context.** The hint is a `Sense` *owned by
  the `SRContext`* — it dies with the context, and deleting it yourself crashes.
  So the teardown order is: disable → `Delete()` / `deleteSRContext`. Same on the
  mid-session weave-throw path, which tears the context down early.
- **Expect `E_NOINTERFACE` on fixed-lens panels** and treat it as normal, not an
  error. `SwitchableLensHint::create` throws there; SR-lib latches that after the
  first attempt so it costs one failed create rather than one per frame.

A useful side effect for debugging: the enable/release transitions are the
cheapest proof that your context lifetime is behaving. If you see the lens
enabled and never released across a mode switch, your release path isn't
running.

**When the process that held the lens is gone.** A game that crashes or is
killed while weaving never releases its hint, and the panel stays lensed over
the desktop afterwards. Seen with a DX12 proxy in Helldivers 2.
What did **not** help: a fresh client creating its own context and calling
`disable()` (its preference is ORed with the stale one), and restarting the
`SRSession.exe` tray process. What did: restarting the **`SR Service`** Windows
service (`Restart-Service -Name 'SR Service' -Force`, elevated). Eye tracking
and `SRSession` keep running across it. Check the lens afterwards. A ten-line
checker is enough: `SRContext::create`, `SwitchableLensHint::create`,
`initialize`, print `isEnabled()`. Pitfall #19 explains why you don't poll it.
Whether a service restart also clears the wedged state in pitfall #14 is
untested; try it before restarting Windows.

## A.11 Ghost reduction — tune the weaver's cancellation, then fix the clipping in your shader

The weaver applies anti-crosstalk for you, using factors measured for the
specific panel, and you never have to enable it. Ghost reduction has two layers,
and you want both:

1. **The weaver's own cancellation** — how hard the panel pre-subtracts the
   opposite view. Tunable, though not where you'd look (A.11.1).
2. **The clipping that cancellation causes** — fixed in your compose shader
   (A.11.2, technique in `output3d` §3.4).

### A.11.1 Tuning the weaver's own anti-crosstalk

`setACTMode`, `setCrosstalkStaticFactor/DynamicFactor` and `setContrast` exist
only on the deprecated `DX9/10/11/12WeaverBase` classes, never on the
`IDX*Weaver1` interfaces that `SR::Create*Weaver` returns — identical in SDK
1.34.10 and 1.36.2, so a newer SDK doesn't add them. **But the runtime still
implements them**, behind a weaver settings interface that every weaver hands
out through `IQueryInterface::query<>()` (all five `I*Weaver1` interfaces derive
from `IQueryInterface`):

```cpp
SR::IWeaverSettings1* s = weaver->query<SR::IWeaverSettings1>();   // nullptr if not offered
```

No SDK version ships the declaration (it lived in an `sr/weaver/IWeaverSettings.h`
that neither 1.34.10 as vendored nor 1.36.2 contains), so it is declared by
hand. **SR-lib's `SR.hpp` already does this** — use it rather than writing your
own:

- Through `SRInterfaceXxx`: `SetCrosstalk(SRCrosstalkMode mode, float strength, float contrast = 1.0f)`.
- With a raw SDK weaver: `SRQueryWeaverSettings(weaver)` →
  `SRReadCrosstalkDefaults(settings, defaults)` once per weaver →
  `SRApplyCrosstalk(settings, defaults, mode, strength, contrast)`.

Prior art: SR Loom (effcol/SR-Loom, v3.1, `SRWeaver.cpp`) drives these on its
standard DX11/DX12 weavers against the 1.36.2 SDK; [VRto3D](https://github.com/oneup03/VRto3D) does it through the
SR-lib helpers on its raw `IDX11Weaver1`, verified on a panel 2026-10-11.

**The ABI caveat — know what you're shipping.** `query<>()` looks the interface
up in the runtime by `typeid` (MSVC matches the decorated name across DLLs), and
every call goes through the *runtime's* vtable. So the namespace (`SR`), class
name (`IWeaverSettings1`), the `virtual IQueryInterface` base and the method
**order** must match exactly:

```cpp
namespace SR {
class IWeaverSettings1 : virtual IQueryInterface {
protected:
    IWeaverSettings1() = default;
    virtual ~IWeaverSettings1() = default;
public:
    virtual void setContrast(float contrast) = 0;
    virtual float getContrast() const = 0;
    virtual void setACTMode(WeaverACTMode mode) = 0;      // global enum, WeaverTypes.h
    virtual WeaverACTMode getACTMode() const = 0;
    virtual void setCrosstalkStaticFactor(float factor) = 0;
    virtual float getCrosstalkStaticFactor() const = 0;
    virtual void setCrosstalkDynamicFactor(float factor) = 0;
    virtual float getCrosstalkDynamicFactor() const = 0;
};
}
```

A runtime that doesn't know the name returns `nullptr` — handle it by hiding the
controls. A future runtime that knew the name but reordered methods would call
the wrong one, and no `try`/`catch` catches that. It is a deliberate, observable
risk (log the defaults you read; garbage there is the tell), not a reason to go
without the controls.

**How to drive them:**

- **Settings belong to one weaver.** A re-created weaver starts from the panel's
  defaults again, so re-read defaults and re-apply after every `Create*Weaver`.
  Setting them on a throwaway helper weaver does nothing to your real one.
- **Read the panel's defaults before changing anything**, and express strength
  and contrast as multiples of them — `1.0` is then "the display as shipped" and
  `(Default, 1.0, 1.0)` restores it exactly. Panel observed 2026-10-11: mode
  `Dynamic` (2), static factor `0.0128535`, dynamic factor `0`, contrast `1.0`.
  Note the dynamic factor reads `0` while Dynamic is the active mode, yet scaling
  strength still visibly changes ghosting — scale **both** factors rather than
  guessing which one the mode consumes. (Reconfirmed in Dynamic mode on DX12 by
  [dusklight3D](https://github.com/oneup03/dusklight3D), 2026-10-19. The zero dynamic factor is **not** why a strength
  control looks dead. That theory cost a debugging round, so don't chase it.
  See the next point.)
- **If Strength appears to do nothing, check `setShaderSRGBConversion`'s
  `read` flag first.** The calls are accepted and logged, but nothing changes on
  the panel. The cause is weaving display-encoded values with `read = false`
  (B.2 gamma note). The cancellation subtracts a linear-light fraction, about
  1.3% of the opposite view, and applying it to encoded values barely moves
  dark tones:

  | 0.20-encoded pixel beside a bright one | `read = true` (linear) | `read = false` on encoded data |
  |---|---|---|
  | strength 1.0× | → ~0.15 | → ~0.19 |
  | strength 3.0× | → 0.00 (clips) | → ~0.16 |

  The tell is comparing integrations: on the same panel, through the same SR-lib
  commit, [VRto3D](https://github.com/oneup03/VRto3D) and [UU3D](https://github.com/oneup03/UU3D) responded and dusklight3D didn't. Their call order
  and timing were identical. The only difference was this flag.
- **Ask for the interface again on every call** instead of caching the pointer —
  `query<>` is cheap, and a weaver destroyed and re-created at the same address
  would otherwise leave you a dangling pointer. Cache only the defaults.
- **Apply on the render thread, only when a value changes** (just before
  `weave()` is a good spot), and log each change — that log is the proof the
  runtime accepted it.
- **Keep them independent of any compose-shader on/off toggle.** These are the
  display's settings, not part of your shader pass. Gating them on the shader
  toggle (pushing the panel defaults while it's off) made the controls silently
  inert while the box was unchecked — confusing, and it was reverted.
- **Persist them per profile like any other setting**; clamp hand-edited values
  to the UI's ranges before applying.

**What each control does, and where to start** (tested values, 2026-10-11):

| Control | Range worth exposing | Start at | Effect |
|---|---|---|---|
| Mode | Display Default / Off / Static / Dynamic | Display Default | Dynamic follows the tracked eye position; Static is a fixed correction; Default is the panel's own choice (Dynamic on current panels) |
| Strength | `0..3×` the panel's factors | **1.30** | Higher cancels more ghosting but clips more in dark areas — pair it with black lift (A.11.2) |
| Weaving contrast | **`0.5..1.0`** | **0.97** | Behaves almost exactly like the compose-shader contrast squeeze. Above `1.0` is not useful |

For the compose-shader levers on top of those, a contrast squeeze of **0.95**
was a good start (tighter than `output3d`'s generic `0.90`, because the weaver's
own contrast is already doing part of the work), plus a small black lift.

### A.11.2 The clipping that's left — fix it in your compose shader

**Residual ghosting on game content is expected, and it is not a sign the
correction is missing.** Cancellation pre-subtracts part of the opposite view,
which pushes values past the ends of the range where your render target clamps
them; the clamped part is what you still see. Desktop and photo content rarely
reaches that. A bright HUD or subtitles over a dark scene reaches it constantly.
Raising the weaver's strength (A.11.1) makes it reach it *more*, which is why
the two layers go together.

**The fix belongs in your compose shader, not here** — see **`output3d` §3.4**,
which owns the technique in full: the two range-compression levers (contrast
squeeze and black lift), why localizing or time-adapting them both fail, and the
cbuffer/string-literal traps. Every consumer that reaches `SetInputTexture` just
rendered that side-by-side texture with a shader, so it is a couple of lines
there — far cheaper than the extra full-screen pass a wrapper would need, and it
keeps SR-lib free of shaders, intermediate textures and pipeline-state
save/restore.

The one part that is genuinely SR-specific is **which space to do it in**.
`output3d` §3.4 assumes linear light; here that is decided by your
`SetShaderSRGBConversion(read, …)` choice (A.2):

| `read` | Cancellation runs in | What you write |
| --- | --- | --- |
| `true` | linear light (your input is linearised first) | the code as published — pivot `0.5` in linear, via a plain `pow(2.2)` |
| `false` | your values as-is | drop both `pow()` calls, pivot `0.5` in your own encoding |

Match the plain `pow(2.2)`, **not** the piecewise sRGB curve — being more
"correct" than the runtime just desynchronises the two. A linear `0.5` pivot is
bright (~`0.73` sRGB-encoded), so lowering contrast lifts blacks noticeably; that
is expected, not a bug.

**When one compose shader serves every output mode**, scope the linear wrap to
the pass that feeds the weaver rather than switching the whole shader. Flipping
`read` to `true` has to bring the shader along with it, but other modes (SbS,
anaglyph, interlaced) have sliders users already calibrated in encoded space.
[dusklight3D](https://github.com/oneup03/dusklight3D) (2026-10-19, via [aurora3D `lib/webgpu/leiasr.cpp`](https://github.com/oneup03/aurora3D/blob/Stereo3D-rebased/lib/webgpu/leiasr.cpp) and its compose shader in
`lib/webgpu/gpu.cpp`) repurposed a padding slot in its stereo UBO as a
`ghostLinear` flag, set only for the LeiaSR SbS pass. Every other mode stays
bit-identical. Also expect **Black Lift to be far stronger in linear**: 1% of
linear is ~`0.12` encoded and 2% is ~`0.17`, so a slider whose help text says
"2–5%" for encoded displays should tell LeiaSR users that 1–2% is plenty.

Finally: **the slider looking better does not prove the panel's cancellation is
what improved.** Reducing contrast reduces ghost visibility on its own, so "it
looks better" is equally consistent with "the correction got the headroom it
needed" and "the correction contributes little here". In practice this rarely
changes what you ship — the remap is worth having either way — but don't reason
backwards from it to conclusions about the runtime's internals.

## A.12 Verification checklist

1. **`dumpbin /imports MyExe.exe`** — every non-system, non-CRT DLL must be under "Section contains the following delay load imports", NOT under the top-level imports.

2. **Launch on a clean machine** (no SR runtime installed). The EXE must start without popups, the engine must run normally, and selecting LeiaSR mode must log a "runtime not installed" message and present plain SbS. **No popup, no crash, no first-chance exception.**

3. **Launch on a machine with SR runtime but no SR display.** Selecting LeiaSR must log a `CreateXxxWeaver` failure and fall back to SbS. **No crash.**

4. **Launch on a machine with SR runtime + SR display.** Selecting LeiaSR must produce autostereoscopic 3D visible without glasses on the SR panel, with depth that responds to head movement. If the image is "stuck" (no head-tracking response), the `ctx->initialize()` step ran in the wrong order or was omitted.

5. **Render-resolution sweep.** Render at well below the panel's native resolution (e.g., 1080p SbS on a 4K SR panel). Weave output must still fill the panel cleanly. Match resolutions for the sharpest output, but lower-than-display is not broken — it's the supported case.

6. **Mid-session display unplug.** Unplug the SR display while the game is running with LeiaSR active. The next `weave()` should throw, get caught, log a warning, and subsequent frames should fall back to SbS. The engine must continue running.

7. **Switchable-lens release test** (switchable-lens panels only). With LeiaSR active, switch to another output mode, then quit. The lens must go off at *each* transition — log the `S_OK` transitions and confirm you see a release, not just an enable (A.10). Leaving it on is invisible in your own window and very visible on everyone else's.

8. **When it's mono but nothing is wrong: prove the weave, don't guess.** `weave()` returning cleanly proves nothing — the runtime degrades to a flat left-eye view rather than failing. Log, once per weaver creation: backbuffer vs window-client vs panel dimensions (all three must match, or the present-time rescale destroys the per-physical-pixel lenticular pattern); `GetPredictedEyePositions` across successive reports (values that never move = nothing is being tracked); and `Dimenco::Weaver::GetBehaviorWhenNotTracking()` (`ShowLeft` / `ShowLeftWithShader` = the runtime is *deliberately* showing you one eye). If all three look healthy and it's still mono, see pitfall #14 — restart Windows.

9. **Weaver anti-crosstalk controls** (A.11.1). On creating the weaver, log whether `query<SR::IWeaverSettings1>()` returned non-null and, if so, the defaults you read (mode, static/dynamic factor, contrast) — sane small factors and contrast `1.0` are the sign the hand-declared interface lines up with the runtime. Change mode and strength and confirm both a logged apply and a visible change on high-contrast content. Sweep strength 100% → 300% in Display Default and watch dark tones beside bright edges: they must visibly change. A logged apply with no visible change means `read` is wrong (pitfall #16), not that the controls are broken. Then re-create the weaver (mode switch) and confirm the settings are re-applied to the new one. On a runtime that returns `nullptr`, the controls must be hidden and nothing else affected.

10. **Ghost-reduction sanity** (A.11.2, technique in `output3d` §3.4). Find high-contrast content — white subtitles or a bright HUD over a dark scene — and try each lever alone. Black lift is the one aimed at what actually clips here, so on a cancelling panel expect it to reach a given ghost reduction at less visible cost than the contrast squeeze; if it does nothing while contrast still helps, the ghosting is not clipping-driven on that panel and contrast is working purely by shrinking the inter-eye difference. Confirm the no-op defaults (`1.00` / `0.00`) are bit-exact, that both are skipped under HDR, and that a 3D screenshot taken with them active is captured without them.

11. **Scaled-display DPI test.** Set the SR panel (or whichever monitor the window occupies) to >100% Windows display scale — e.g. 150% — and launch. The weave must still land 1:1 on panel pixels: clean autostereo, no DPI-factor zoom/crop, no blur. If the image looks zoomed by exactly the scale factor, the process isn't per-monitor-DPI-aware (A.2.2) and/or the SR service's one-shot resize wasn't corrected (pitfall #11). Log the runtime awareness readback and confirm it says per-monitor.

12. **32-bit Unity game** (x86 builds only). Launch any 32-bit Unity title with weaving on. The log must show the weaver created, the first woven frame, and your handler's "Virtual PC probe answered" line, and the game must keep responding for a minute or more. A hang right after context creation means the probe handler is missing or was installed too early (A.13.1).

13. **Kill test.** Kill the host process (Task Manager) while it weaves, and look at the desktop. If it stays lensed, that is the runtime's behaviour, not yours. Know the service restart that clears it (A.10, last paragraph) before a user reports it.

14. **Exit test.** Close the host from its own UI while weaving, several times. The process must actually end (no windowless instance left in Task Manager), and the tracking camera must turn off. A process that outlives its window with the camera on is a weaver destroyed on the wrong thread (pitfall #20).

15. **sRGB back buffer test** (when you copy the game's back buffer). Render a flat mid-grey (128) into an sRGB back buffer and read the woven screen pixel back with `GetPixel` from a DPI-aware process. It must still read 128; a double gamma encode read 188 (A.13.3).

16. **Input test when hosted next to another wrapper.** After weaving starts, keyboard, mouse and controller must still reach the game. Compare a menu screenshot before and after a key press. Lost input means your code created a window or device that the other wrapper subclassed (A.13.2).

---

## A.13 Hosting the weaver in a process you don't own

*Added in the nautymac fork. Everything here comes from integrations that weave inside somebody else's process, verified on an Acer SpatialLabs panel (SR Platform 1.34.10): a `dxgi.dll` proxy that weaves geo-11/3DMigoto's SBS output (geo-11_SR SRWeave, x64 and x86), hooks behind TriDef 3D (D3D9 and DX11), drop-in `opengl32.dll` / `vulkan-1.dll` / `d3d12.dll` proxies (OGL3D, VK3D, DX12_3D), and bridge DLLs in two emulators (Xenia-3D, Azahar-SR). The `api-proxy-stereo` skill covers how those proxies make the stereo pair, and `wrapper-injection` how such a DLL gets loaded at all (its §5 lists the SR traps below in short form). This section is only about the weaver.*

When the weave runs in a game you didn't write, A.1–A.12 still hold. In addition, the host's exception handlers, the other wrappers in the process, its DPI awareness and its GL state all act on the weaver.

### A.13.1 32-bit hosts: answer the runtime's Virtual PC probe first

While the weaver is being created, the 32-bit runtime (`DimencoWeaving32.dll`, SR Platform 1.34.10, RVA `0x3e1c6`) checks whether it runs inside Microsoft Virtual PC. It executes the "vpcext" instruction `0F 3F 07 0B` under its own SEH frame. On real hardware that raises `EXCEPTION_ILLEGAL_INSTRUCTION`, and the DLL's handler sets `EBX = 0xFFFFFFFF` ("not Virtual PC") and skips the four bytes.

**Vectored exception handlers run before any SEH frame.** A host whose VEH takes an illegal instruction for a crash never lets the DLL see it. Mono's VEH does exactly that, and every 32-bit Unity game built with the Mono scripting backend has it (IL2CPP builds were not tested):

- **With Unity's render thread:** the render thread stays inside `mono.dll` and the main thread waits for it. Windows reports *not responding* (AppHangB1).
- **With single-threaded rendering** (`-force-gfx-direct`): the game dies with `0xC0000409`.
- **In your log:** it ends after the SR context was created, with no "weaver created" line.

The case found was Absolute Drift (Unity 2017.4, x86). A 32-bit non-Unity game (Sine Mora EX, D3D9 through dgVoodoo) wove fine, and the x64 runtime DLL contains no such probe. It was diagnosed from a 32-bit minidump of the hung game (take it with a 32-bit tool, e.g. `procdump -ma` or `MiniDumpWriteDump` from a 32-bit process): the exception record was `0xC000001D` at `DimencoWeaving32.dll+0x3e1c6`, and the stuck thread was in `mono.dll`.

**Fix:** put a handler in front of the host's that does exactly what the DLL's own handler does, for that one instruction only (from [geo-11_SR `SRWeave/src/srweave_geo11.cpp`](https://github.com/nautymac/geo-11_SR/blob/master/SRWeave/src/srweave_geo11.cpp); the OGL3D-SR and tridef3d-win11-injector x86 builds carry the same code):

```cpp
#ifndef _WIN64
std::atomic<int> g_vpcProbes{0};

LONG CALLBACK VpcProbeHandler(EXCEPTION_POINTERS* ep)
{
    if (ep->ExceptionRecord->ExceptionCode != EXCEPTION_ILLEGAL_INSTRUCTION) return EXCEPTION_CONTINUE_SEARCH;
    const unsigned char* ip = (const unsigned char*)(ULONG_PTR)ep->ContextRecord->Eip;
    __try {
        if (ip[0] != 0x0F || ip[1] != 0x3F || ip[2] != 0x07 || ip[3] != 0x0B) return EXCEPTION_CONTINUE_SEARCH;
    } __except (EXCEPTION_EXECUTE_HANDLER) {
        return EXCEPTION_CONTINUE_SEARCH;
    }
    ep->ContextRecord->Ebx = 0xFFFFFFFF;   // "not Virtual PC", what the DLL's own handler sets
    ep->ContextRecord->Eip += 4;           // skip 0F 3F 07 0B
    g_vpcProbes++;
    return EXCEPTION_CONTINUE_EXECUTION;
}
#endif

// installed at the first SR call (the game's own handlers, Mono's included, are registered by then)
void InstallVpcProbeHandler()
{
#ifndef _WIN64
    static bool done = false;
    if (done) return;
    done = true;
    AddVectoredExceptionHandler(1, VpcProbeHandler);
#endif
}
```

Details that matter:

- **Install it at the first SR call, not in `DllMain`.** `AddVectoredExceptionHandler(1, …)` puts the handler at the head of the list *as it is at that moment*. A handler the game registers later with `First = 1` goes in front of yours again. Mono has registered its handler long before your first `SRContext::create()`.
- **Match the exact bytes**, and read them under `__try`. Any other illegal instruction is still the host's business.
- **Log the count** after weaver creation (SRWeave: *"SR runtime Virtual PC probe (illegal instruction 0F 3F 07 0B) answered ahead of the game's exception handlers (1 so far)"*). A user's log then shows the handler ran.
- **Harmless first-chance lines.** In a host whose handlers leave the probe alone, a first-chance exception logger shows `0xC000001D at DimencoWeaving32.dll+0x3e1c6` once at start-up, together with a `0xC0000096` (privileged instruction) first chance just before it. The DLL handles both, and games ran normally with them (Amnesia, Broken Age through OGL3D). Don't chase them.
- **The x86 runtime DLLs have their own names and folder**: `SimulatedRealityCore32.dll`, `SimulatedRealityDisplays32.dll`, `SimulatedRealityDirectX32.dll`, … under `C:\Program Files (x86)\…\Platform\bin`. Your delay-load list (A.3.1) and runtime lookup need the 32-bit names. The x86 SDK import libraries were not in the win64 SDK package used here: they came from the win32 runtime installer bundled inside it (unpacked with 7-Zip). The 32-bit `dxgi.dll` exports also need their own `.def`, because the ordinals differ from x64.

Verified in game with SRWeave: Absolute Drift wove and stayed responsive; its log shows the probe answered between context and weaver creation, and the first woven frame 40 ms later. Before the fix it hung every time. The OGL3D and TriDef x86 builds carry identical code, but only SRWeave's was tested in a Unity game.

### A.13.2 Next to another wrapper: no throw-away objects, hook the real factory

The usual way for a hook DLL to find `Present` is to create a dummy window, device and swap chain at start-up and read their vtables. **Inside 3DMigoto/geo-11 that broke the game.** geo-11 wraps and subclasses every device, swap chain and window it sees. The dummy window's subclassing replaced the game window's procedure inside geo-11, and keyboard, mouse and controller input stopped reaching the game.

What works (SRWeave, a `dxgi.dll` proxy loaded by geo-11's `d3d11.dll`):

1. On the first call into your `dxgi.dll` exports, create a factory from the **real** System32 `dxgi.dll` (`CreateDXGIFactory1`). A factory needs no window.
2. Hook its `CreateSwapChain` (vtable slot 10) and `CreateSwapChainForHwnd` (`IDXGIFactory2`, slot 15).
3. When the game's swap chain comes back, hook `Present` (8), `ResizeBuffers` (13) and `Present1` (`IDXGISwapChain1`, 22) on the **real** swap chain.
4. Hook a target only if it lies inside System32 `dxgi.dll` (`d3d9.dll` for D3D9). Otherwise log it and skip: you would be hooking another wrapper's code.

**The object you are handed may be a wrapper.** Because SRWeave hooks the *real* factory, the swap chain it sees is normally the real one: in every geo-11 game it wove, no unwrapping was needed. Behind TriDef 3D it was. TriDef's DXGI layer (`TriDefDXGI64.dll`) returns the swap chain as a complete wrapper, and so does TriDef's D3D9 device (a temporary device whose `GetSwapChain(0)` even failed with `D3DERR_INVALIDCALL`). To find the real object inside one:

- Scan the wrapper's pointer-sized fields for an object whose vtable mostly points into the system DLL: at least 12 of the first 18 slots for `IDXGISwapChain`, at least 80 of 119 for `IDirect3DDevice9`.
- Confirm it with `QueryInterface`, then hook that object's vtable.
- Read every pointer with an exception-safe read.

In TriDef's DXGI wrapper the real swap chain was field 1. SRWeave keeps the same scan as a fallback for wrappers that hand out wrapped swap chains.

At present time:

- **Order:** your hook runs inside the real `Present`, so after the other wrapper has composed its SBS picture into the back buffer. Copy the back buffer, weave into it, then call the original `Present`.
- **Re-entry:** guard against it with a thread-local flag, because the weave's own work can come back through the wrappers. Skip `DXGI_PRESENT_TEST`.
- **State:** save and restore the immediate context's state around your copy and weave (D3D9: a `D3DSBT_ALL` state block plus render target, depth surface and viewport). The game's next draw must not see yours.
- **D3D9 `Reset`:** call `invalidateDeviceObjects()` before it and `restoreDeviceObjects()` after (B.5).

### A.13.3 sRGB back buffers: typeless copy, one view format for input and output

When you weave a copy of the game's back buffer, the back buffer's format decides two things:

- **The copy must be typeless.** A typed `*_UNORM_SRGB` texture accepts views of exactly that format. With any other view format `CreateShaderResourceView` failed with `E_INVALIDARG` every frame, so weaving never started (art of rally, Unity, exclusive fullscreen, `R8G8B8A8_UNORM_SRGB` back buffer). `CopyResource` from the typed back buffer into a typeless texture of the same family is legal.
- **The input view and the output view must agree.** An sRGB back buffer with a UNORM input view and an sRGB output RTV encodes gamma twice, and the picture is too bright (Red Faction Guerrilla Re-Mars-tered). Swap-chain buffers accept the sibling UNORM/sRGB view, so create the output RTV in the **same** format as the input SRV. Fall back to the back buffer's own format only if the sibling view is refused, and log a warning.

```cpp
// copy texture: TypelessOf(backBuffer.Format)            e.g. R8G8B8A8_UNORM_SRGB -> R8G8B8A8_TYPELESS
// view format vf: auto = follow the back buffer          sRGB back buffer -> *_UNORM_SRGB, else *_UNORM
D3D11_TEXTURE2D_DESC td = bd;
td.Format    = TypelessOf(bd.Format);
td.BindFlags = D3D11_BIND_SHADER_RESOURCE | D3D11_BIND_RENDER_TARGET;
dev->CreateTexture2D(&td, nullptr, &copy);
D3D11_SHADER_RESOURCE_VIEW_DESC sd = {};  sd.Format = vf;  sd.ViewDimension = D3D11_SRV_DIMENSION_TEXTURE2D;  sd.Texture2D.MipLevels = 1;
dev->CreateShaderResourceView(copy, &sd, &srv);          // weaver input
D3D11_RENDER_TARGET_VIEW_DESC rd = {};    rd.Format = vf;  rd.ViewDimension = D3D11_RTV_DIMENSION_TEXTURE2D;
dev->CreateRenderTargetView(backBuffer, &rd, &bbRtv);    // weaver output: same sRGB-ness as the input
weaver->setInputViewTexture(srv, w / 2, h, vf);
```

Verified with a test app that fills the back buffer with flat grey 128: the screen pixel read back 188 with the mismatched views and 128 after the fix. RFG's brightness was then checked by eye. This is independent of pitfall #16, which is about the weaver's own conversion flags. Use both rules together.

### A.13.4 DPI-unaware games

A.2.2 says to make the process per-monitor DPI aware before init. In a game you don't own, choosing that changes the game too. Two options worked (OGL3D):

**(a) Make the whole process aware from the proxy's `DllMain`** (`SetProcessDpiAwarenessContext(PER_MONITOR_AWARE_V2)`, before the game creates its window). The game then sees physical pixels and can render at the panel's native resolution. Amnesia: The Dark Descent ran woven at 3840 × 2160 once its own config asked for that size. This is the simplest option, but the game's UI may become tiny at 4K.

**(b) Keep the game unaware at its own resolution and present yourself.** This is OGL3D's upscale mode:

- The eyes go GL → D3D11 through `WGL_NV_DX_interop2`. A second **GL** window presented from a DPI-unaware process crashed in NVIDIA's DXGI-based GL present.
- They are woven by the **DX11** weaver at the panel's resolution, into your own per-monitor-aware popup over the game window.
- Window details:
  - The window is owned by the game window (so it stays above it without being topmost) and created with `WS_EX_NOACTIVATE | WS_EX_TOOLWINDOW`.
  - Windows may give an owned window its owner's DPI awareness. Check `GetWindowDpiAwarenessContext`, and if it is not per-monitor, recreate the window unowned and `WS_EX_TOPMOST`.
  - It answers `WM_NCHITTEST` with `HTTRANSPARENT` and `WM_MOUSEACTIVATE` with `MA_NOACTIVATE`, so the mouse reaches the game.
  - **`HTTRANSPARENT` only passes input to windows of the same thread**, so create the window on the game window's thread. Under WineD3D (Samorost 2), the frame arrives on the renderer's command thread, which has no message pump. A window created there made the game "not responding" and ignored clicks. The fix (OGL3D-SR `flato3d` branch) creates it from a message posted to the game window's thread (picked up by a `WH_GETMESSAGE` hook) and uses only asynchronous calls across threads (`ShowWindowAsync`, `SWP_ASYNCWINDOWPOS`, posted `WM_CLOSE`).
- **The SR DLLs measure the window themselves.** They do it also from a window procedure they subclass, so on the game's DPI-unaware thread. They got DPI-virtualized sizes, and the weave covered only part of the window. Fix: patch the `user32` imports of every module loaded from the SR runtime folders, so that each geometry call runs per-monitor aware:

```cpp
struct PhysicalDpiScope {   // this thread per-monitor aware for one call: window coordinates in physical pixels
    HANDLE prev = nullptr;
    PhysicalDpiScope()  { if (SetThreadCtx()) prev = SetThreadCtx()((HANDLE)-4 /* PER_MONITOR_AWARE_V2 */); }
    ~PhysicalDpiScope() { if (prev && SetThreadCtx()) SetThreadCtx()(prev); }
};
BOOL WINAPI Phys_GetClientRect(HWND h, LPRECT r) { PhysicalDpiScope s; return GetClientRect(h, r); }
// ... same for GetWindowRect, ClientToScreen, ScreenToClient, MonitorFromWindow, GetMonitorInfoA/W,
//     EnumDisplayMonitors, EnumDisplaySettingsA/W, GetSystemMetrics

// for each loaded module whose path contains \simulated reality\, \spatiallabs\ or \leiasr\ :
//     rewrite its user32.dll import slots (by name, OriginalFirstThunk) to the Phys_ versions
```

  Preload the runtime DLLs and patch **before** creating the weaver, then patch again right after it (it loads more DLLs). The source is OGL3D-SR [`src/present.cpp`](https://github.com/nautymac/OGL3D-SR/blob/main/src/present.cpp) (`PatchSrModules`).
- **The weaver fills an area the size of one input view.** Upscale each eye to the window size first: an input `2 × windowW` by `windowH`, per-eye width `windowW`. Weaving the game-resolution views directly left the rest of the window unwoven.

Verified on Amnesia at 1920 × 1080, DPI-unaware: full-screen weave at 3840 × 2160 with 2× UI. It was checked with a screen capture from a DPI-aware process (`CopyFromScreen`) and by eye on the panel.

### A.13.5 Teardown on threads you don't control

- **Retire weavers to a cleanup thread.** In Azahar-SR, emulation stops on the emulation thread while the GUI thread waits for it. A DX11 weaver's `destroy()` waits for its own processing threads, and those talk to the window's thread, so destroying the weaver there deadlocked the exit (pitfall #20). The fix keeps a list of retired weavers, served by one detached thread that destroys them. When no weaver is live, that thread also calls `SRContext::deleteSRContext`, which turns the eye-tracking camera off while the app stays open. A restart re-creates the context.

```cpp
void CleanupLoop() {                      // one detached thread, started on the first Retire()
    for (;;) {
        std::vector<SR::IDestroyable*> weavers;
        { std::unique_lock lock{g.mutex};
          g.retired_cv.wait(lock, [] { return !g.retired.empty(); });
          weavers.swap(g.retired); }
        for (auto* w : weavers) { try { w->destroy(); } catch (...) {} }
        std::scoped_lock lock{g.mutex};
        if (g.ctx && !g.gl_weaver && !g.gl_dropped && !dx.weaver && g.retired.empty()) {
            try { SR::SRContext::deleteSRContext(g.ctx); } catch (...) {}
            g.ctx = nullptr; g.lens = nullptr;   // the lens hint died with the context (A.10)
        }
    }
}
// Stop path (caller holds g.mutex): lens off first, then Retire(dx.weaver); dx.weaver = nullptr;
```

- **A GL weaver whose context isn't current on the stopping thread: drop it, don't destroy it.** Its GL objects belong to the presentation context. Keep the SR context too while a dropped GL weaver may still use it. The camera then stays on until the process exits, which is acceptable; a crash is not.
- **Bridge DLLs.** When the host can't link the proprietary SDK (Azahar is GPL) or uses another CRT (Xenia is `/MT`), put the SDK in a small MSVC bridge DLL with a C API. Examples: `AzSr_Weave` / `AzSr_Dx11Weave` / `AzSr_Stop`, and `xenia3d_sr_weave`. The bridge delay-loads the runtime (A.3), and the host loads the bridge with `LoadLibrary`. This is the same shape as B.1's C-ABI shim.

### A.13.6 Diagnostics and testing

- **The SR platform keeps per-application logs** under `C:\ProgramData\Simulated Reality\Applications\<app>\Log`: lens enable/disable, "weaving is (not) needed", occlusion. `AQAUserPS.exe` reports that it "occludes" every game; that is harmless.
- **Log one line per stage**: context created, weaver created, first woven frame (with sizes), lens on/off, probe answered (A.13.1), back buffer differing from the window client (pitfall #23). A user's log then shows where it stopped.
- **Test remotely in SBS.** A woven picture is unreadable over remote desktop or streaming, and in screenshots viewed anywhere but the panel. When you or a tester judge the picture remotely, switch the output to plain SBS, and switch weaving back on at the panel. Automated checks should look at the pre-weave SBS frame, not the woven screen.
- **Verify presentation with a capture taken from outside**: a DPI-aware `CopyFromScreen` or DXGI desktop duplication. In-DLL probes see what you rendered, not what the compositor shows; a DPI-unaware capture shows a scaled picture.

---

# Part B — Per-backend specifics

## B.1 OpenGL

| Thing | Concrete value |
|---|---|
| Weaver type | `SR::IGLWeaver1*` |
| Create function | `SR::CreateGLWeaver(SRContext&, HWND, IGLWeaver1**) → WeaverErrorCode` |
| Header | `<sr/weaver/glweaver.h>` |
| CMake | `SRLib::SR` via A.0's package; longhand is `find_package(srOpenGL REQUIRED)` against a full SDK install |
| Link target | `srOpenGL::srOpenGL` |
| Texture handle type | `GLuint` (raw GL texture name) |
| Texture-format token | `GLenum` — pass `GL_RGB` (0x1907), `GL_RGBA` (0x1908), etc., matching your FBO's color attachment format |
| Output destination | currently-bound GL framebuffer at the current GL viewport |
| Filter setting for upscale | `GL_LINEAR` for `GL_TEXTURE_MIN_FILTER` and `GL_TEXTURE_MAG_FILTER` on the source texture |
| Delay-load DLL list | the A.3.1 list with `SimulatedRealityOpenGL.dll` (the GL backend DLL) |

Worked end-to-end example: RingRacers-3D's [`src/hwr2/stereo_leiasr.cpp`](https://github.com/oneup03/RingRacers-3D/blob/master/src/hwr2/stereo_leiasr.cpp), driven from [`src/i_video_common.cpp`](https://github.com/oneup03/RingRacers-3D/blob/master/src/i_video_common.cpp)'s `I_FinishUpdate` present path.

### Getting the GL texture name from an RHI / abstraction layer

If your engine wraps GL behind an RHI (e.g. RingRacers-3D's HWR2 layer), there will be a "native handle" accessor. Example:

```cpp
const uintptr_t tex_id = rhi->get_native_texture(stereo_backbuffer->color());
if (tex_id != 0) {
    R_LeiaSR_Weave(static_cast<uint32_t>(tex_id),
                   stereo_backbuffer_width, stereo_backbuffer_height,
                   GL_RGB);
}
// Some RHI backends (non-GL) may return 0 — fall back to plain SbS in
// that case rather than crashing.
```

If the native-handle accessor returns 0 (e.g. you're running a Vulkan or DX RHI backend through the same code path), that's a clean signal to fall back to SbS: the GL weaver can't operate on a non-GL texture.

### Full-SbS via present-time blit-stretch (Option 2 from A.1)

When the engine's stereo render produces a half-SbS-packed `W × H` backbuffer (the normal output of a per-eye loop that places LEFT in the left half and RIGHT in the right half of one backbuffer-sized FBO), the cleanest way to feed the weaver a `2W × H` full-SbS source is a **present-time blit-stretch** into a wider intermediate texture. No re-render, no engine-side stereo plumbing changes — purely a present-path addition.

Trade-off vs. native 2W × H rendering: the *signal* is still `W/2 × H` per eye (the blit is a 2× bilinear upscale), but the weaver receives a 2W × H input and operates without its own internal upscale. For most current Leia panels (where the lenticular optical splitter delivers half the panel column count per eye anyway), the visible quality is identical to native 2W × H. For newer full-rate Leia panels, native 2W × H wins — see the note at the end of this section. Start with blit-stretch; revisit later if your target panel benefits.

The RingRacers-3D HWR2 reference does it like this (the same `BlitRectPass` it uses for normal mono present is reused for the stretch):

```cpp
// In the present path, when stereo mode is LeiaSR. Assumes we're currently
// in the default render pass (SDL window).
const uint32_t target_w = 2u * static_cast<uint32_t>(backbuffer_width);
const uint32_t target_h =      static_cast<uint32_t>(backbuffer_height);

// (Re-)allocate the wide intermediate if needed.
if (!leiasr_source_color
    || cached_leiasr_w != target_w
    || cached_leiasr_h != target_h)
{
    if (leiasr_source_color) rhi->destroy_texture(leiasr_source_color);
    rhi::TextureDesc d {};
    d.format = rhi::TextureFormat::kRGB;
    d.width  = target_w;
    d.height = target_h;
    d.u_wrap = rhi::TextureWrapMode::kClamp;
    d.v_wrap = rhi::TextureWrapMode::kClamp;
    leiasr_source_color = rhi->create_texture(d);
    cached_leiasr_w = target_w;
    cached_leiasr_h = target_h;
}

// 1. Pop the default render pass so we can target the wide intermediate.
rhi->pop_render_pass();

// 2. Push render pass to leiasr_source. Clear-on-load is fine; the blit
//    overwrites every pixel anyway. No depth attachment — it's a 2D blit.
rhi::RenderPassBeginInfo info {};
info.color_attachment = leiasr_source_color;
info.color_load_op    = rhi::AttachmentLoadOp::kClear;
info.color_store_op   = rhi::AttachmentStoreOp::kStore;
info.clear_color      = {0.f, 0.f, 0.f, 1.f};
rhi->push_render_pass(info);

// 3. Force LINEAR sampling on the source — without it the 2× stretch
//    aliases visibly. correct_aspect=false on the blit_rect output makes
//    the W × H source fill the full 2W × H target rather than letterbox.
rhi->update_texture_settings(backbuffer_color,
    TextureWrapMode::kClamp, TextureWrapMode::kClamp,
    TextureFilterMode::kLinear, TextureFilterMode::kLinear);
blit_rect->set_texture(backbuffer_color, backbuffer_width, backbuffer_height);
blit_rect->set_output(0, 0, target_w, target_h,
                      /*correct_aspect=*/false, /*flip=*/true);
blit_rect->draw(*rhi);

// 4. Pop the intermediate render pass.
rhi->pop_render_pass();

// 5. Re-push the default render pass for the weave.
rhi->push_default_render_pass(/*clear=*/true);

// 6. Weave from the full-SbS intermediate.
const uintptr_t tex_id = rhi->get_native_texture(leiasr_source_color);
R_LeiaSR_Weave(static_cast<uint32_t>(tex_id),
               static_cast<int>(target_w),
               static_cast<int>(target_h),
               GL_RGB);
```

Key points:

- The blit pass's `correct_aspect = false` is what makes it stretch — with the default `correct_aspect = true` you'd get black bars instead of a fill, and the half-half SbS split wouldn't align with the weaver's `uv.x = 0.5` boundary.
- `LINEAR` sampling on the source is mandatory; the 2× upscale at `NEAREST` aliases visibly.
- Pop and re-push the default render pass around the intermediate work — otherwise `weave()` writes into the intermediate instead of the SDL window.
- The intermediate texture is allocated lazily and resized when the backbuffer dimensions change, same pattern as the engine's own backbuffer.
- The fallback path (when `get_native_texture` returns 0, e.g. on a non-GL RHI backend) should bypass the intermediate entirely and just present the original `W × H` backbuffer to the window — there's no point doing a wide blit if we're not feeding the weaver anyway.

### When to upgrade to native 2W × H instead

A truly native 2W × H render (each eye genuinely produced at W × H by the engine's per-eye render loop, no resampling anywhere) requires touching the engine's stereo plumbing in ~5 places: the backbuffer FBO width, the per-eye viewport math, the post-processing composite output viewports, the 2D HUD layer's ortho/viewport, and the present-time blit setup. None of it is hard in OpenGL — wider FBOs and shifted viewports are GL 101 — but the change is no longer localized to the present path.

It's worth doing if:

- Your target Leia panel is one of the newer **full-rate** designs where each eye can receive full panel resolution (lenticular optical splitters that *don't* discard half the columns per eye). On those, the blit-stretch upscale visibly loses detail vs. a native render.
- You're already paying for a 2× GPU bill (e.g. shipping a game where the host already targets very-high-res displays) and the wider FBO is the natural next step.

For older / mainstream Leia panels with a 2× lenticular splitter, the blit-stretch is visually equivalent to native — the optics discards the extra columns anyway — and the GPU savings are meaningful.

### Hosting in a non-MSVC build: the C-ABI shim DLL pattern

The default B.1 setup (link SR-lib's `SRLib::SR` — or, against a full SDK install, `find_package(simulatedreality)` + `find_package(srOpenGL)` — into the engine binary and apply `/DELAYLOAD` to the result) **assumes the host engine is built with MSVC**. If your engine is built with **MinGW64** — or any toolchain whose C++ ABI doesn't match MSVC — that doesn't work. The SR SDK is MSVC-compiled and the OpenGL weaver is exposed as `SR::IGLWeaver1`, a C++ class with **virtual inheritance** (`public virtual IDestroyable, public virtual IQueryInterface, public virtual IWeaverBase1`). The MSVC C++ ABI for virtual inheritance lays out v-pointers, vbtables and adjuster fields differently from the Itanium ABI MinGW follows, so MinGW can't call into the SR libraries directly — symptoms range from link errors to silent vtable misalignment / corrupt `this` pointers at runtime.

> ⚠ This is **not** something `extern "C"` declarations or a `.def` file can paper over. The mismatch is in the v-pointer layout the host engine would need to chase to dispatch virtual calls. The C interface (`<sr/*_c.h>`, `<sr/weaver/WeaverC.h>`) on its own *does* link cleanly into MinGW — the MinGW x64 `ld` can read MSVC `.lib` import libs — but those C surfaces don't expose `IGLWeaver1`. The GL weaver is C++-only as of SDK 1.34.

The standard fix is a **C-ABI shim DLL**: a small MSVC-compiled translation unit that owns the C++ side and exposes a flat C ABI the MinGW engine can `LoadLibrary` at runtime. Only types both ABIs agree on (POD ints, `void*`, raw GL texture ids) cross the boundary; no SR symbols ever appear in the engine's link line.

```
host engine (MinGW)                  leiasr_shim.dll (MSVC)
─────────────────────                 ──────────────────────
R_LeiaSR_Init(hwnd) ──── LoadLibraryA("leiasr_shim.dll")
                         GetProcAddress("srk_init"/"srk_weave"/"srk_shutdown")
                         srk_init(hwnd) ──────────────────► SR::SRContext::create()
                                                            SR::CreateGLWeaver(...)
                                                            g_ctx->initialize()
R_LeiaSR_Weave(tex,w,h)  srk_weave(tex,w,h) ──────────────► g_weaver->setInputViewTexture(...)
                                                            g_weaver->weave()
R_LeiaSR_Shutdown()      srk_shutdown() ──────────────────► g_weaver->destroy()
                                                            SR::SRContext::deleteSRContext(g_ctx)
```

Two production reference implementations of this pattern, both MinGW host + MSVC shim:

- **[Kart-Public-3D](https://github.com/oneup03/Kart-Public-3D)** — [`leiasr_shim/leiasr_shim.cpp`](https://github.com/oneup03/Kart-Public-3D/blob/master/leiasr_shim/leiasr_shim.cpp) + [`src/r_stereo_leiasr.c`](https://github.com/oneup03/Kart-Public-3D/blob/master/src/r_stereo_leiasr.c). MinGW counterpart to the RingRacers-3D MSVC integration referenced earlier.
- **[perfect_dark_3D](https://github.com/oneup03/perfect_dark_3D)** — [`leiasr_shim/leiasr_shim.cpp`](https://github.com/oneup03/perfect_dark_3D/blob/port/leiasr_shim/leiasr_shim.cpp) + [`port/src/stereo_leiasr.c`](https://github.com/oneup03/perfect_dark_3D/blob/port/port/src/stereo_leiasr.c). Same pattern applied to the Perfect Dark N64 port's MinGW64 build; uses the lazy-init split from A.2.1 (shim DLL loaded eagerly at process start, `srk_init` deferred to the first weave attempt) plus both the `LoadLibraryW` preflight and `/EHa` SEH catch layered together per A.3.3.

#### Designing the shim's C ABI

Keep the surface minimal. Three functions matching the lifecycle from A.2:

```cpp
extern "C" __declspec(dllexport) int  srk_init(void *hwnd);
extern "C" __declspec(dllexport) void srk_weave(unsigned int tex_id, int width, int height);
extern "C" __declspec(dllexport) void srk_shutdown(void);
```

`void*` for HWND (the engine doesn't need `<windows.h>` to call `srk_init`), `unsigned int` for the GL texture id (== `GLuint` == `uint32_t`), plain `int` for dimensions. The texture *format* is hardcoded on the shim side (`GL_RGB8 = 0x8051`) since it always matches the engine's `pglCopyTexImage2D(GL_RGB, ...)` capture — exposing it would require dragging GL headers into the C ABI for no benefit.

The shim owns a single global `SR::SRContext` + `SR::IGLWeaver1`. Multiple windows would need handle-keyed state but for a single-window game globals are fine.

#### Where delay-load + the preflight live

**Inside the shim's CMakeLists.txt, not the engine's.** The shim is the binary that imports SR symbols at link time, so it's where you apply A.3.1's `/DELAYLOAD` block. The engine's import table has no SR-side symbols at all — its only SR-side dependency is `leiasr_shim.dll`, which it loads explicitly via `LoadLibraryA`. If the shim DLL isn't present, `LoadLibraryA` returns `NULL` and the engine falls back to SbS cleanly with no popup.

Same applies to the A.3.2 `LoadLibraryW("DimencoWeaving.dll")` preflight — it lives inside the shim's `srk_init`, gating the actual SR calls. From the engine's perspective, `srk_init` returns 0 if the runtime is missing and that's all it needs to know.

So the shim's CMakeLists.txt block is identical to A.3.1 (every SR DLL + `opencv_world343.dll` + transitive deps under `target_link_options(... /DELAYLOAD:...)`), just applied to `leiasr_shim` instead of the main engine binary. After link, `dumpbin /dependents leiasr_shim.dll` should show only `KERNEL32.dll` / `VCRUNTIME140.dll` / `api-ms-win-crt-*` under top-level imports, with everything SR-side under delay-load imports.

#### Engine-side runtime loader

Pure C, MinGW-compiled. Lazily LoadLibrary's the shim on first use, GetProcAddress'es the three exports, calls the shim's init. Any failure (DLL missing, wrong shim version, SR runtime not installed, no display) ends up as `available = false` and the engine treats LeiaSR mode as "fall back to SbS":

```c
#include <windows.h>

typedef int  (*PFN_srk_init)(void *hwnd);
typedef void (*PFN_srk_weave)(unsigned int tex_id, int width, int height);
typedef void (*PFN_srk_shutdown)(void);

static HMODULE shim;
static PFN_srk_init     p_init;
static PFN_srk_weave    p_weave;
static PFN_srk_shutdown p_shutdown;
static boolean          available;
static boolean          init_attempted;

void R_LeiaSR_Init(void)
{
    void *hwnd;
    if (init_attempted) return;
    init_attempted = true;

    hwnd = I_GetWindowHandle();
    if (!hwnd) { init_attempted = false; return; } // SDL not up yet, retry later

    shim = LoadLibraryA("leiasr_shim.dll");
    if (!shim) return;  // Shim DLL not deployed — silent SbS fallback.

    // Dual cast (PFN_*)(void *)GetProcAddress(...) silences GCC's
    // -Wbad-function-cast warning for the FARPROC-to-typed-pointer cast.
    p_init     = (PFN_srk_init)    (void *)GetProcAddress(shim, "srk_init");
    p_weave    = (PFN_srk_weave)   (void *)GetProcAddress(shim, "srk_weave");
    p_shutdown = (PFN_srk_shutdown)(void *)GetProcAddress(shim, "srk_shutdown");
    if (!p_init || !p_weave || !p_shutdown)
    {
        // Shim present but missing one of the exports — wrong version
        // (a different srk_* surface) or a build that lost an export.
        FreeLibrary(shim);
        shim = NULL;
        return;
    }

    if (p_init(hwnd))
        available = true;
}

boolean R_LeiaSR_Available(void)
{
    if (!init_attempted) R_LeiaSR_Init();   // lazy first-use init
    return available;
}

void R_LeiaSR_Weave(unsigned int tex_id, int width, int height)
{
    if (available && p_weave) p_weave(tex_id, width, height);
}

void R_LeiaSR_Shutdown(void)
{
    if (available && p_shutdown) p_shutdown();
    if (shim) { FreeLibrary(shim); shim = NULL; }
    p_init = NULL; p_weave = NULL; p_shutdown = NULL;
    available = false;
}
```

Call `R_LeiaSR_Shutdown` from the engine's GL-context teardown (e.g. `I_ShutdownGraphics`) **before** the GL context goes away — the SR runtime holds GL resources keyed to that context and dropping it under the runtime can segfault on the next launch.

#### Build setup

The shim is **a separate cmake project**, not wired into the engine's main build (which is GCC/Make):

```
project-root/
├── src/                      ← engine, built with MinGW/Make
├── leiasr_shim/
│   ├── CMakeLists.txt        ← MSVC build, separate from engine
│   ├── leiasr_shim.cpp
│   └── README.md
└── libs/SR-lib/              ← SR SDK (git submodule, api_expansion branch)
```

The shim's CMakeLists.txt is small: `add_subdirectory` the SR-lib submodule, link `SRLib::SR`, build a `SHARED` library, and call `srlib_apply_delayload()` on it (A.0). Against a full SDK install instead, it's A.3.1's shape — `find_package(simulatedreality REQUIRED)` + `find_package(srOpenGL REQUIRED)` plus the explicit `/DELAYLOAD` block. Build is:

```sh
cd leiasr_shim
cmake -B build -A x64                       # configures with Visual Studio generator
cmake --build build --config Release        # produces build/Release/leiasr_shim.dll
```

The engine's build script (or CI workflow) stages the resulting DLL next to the engine `.exe`. If the shim hasn't been built, the engine still builds and runs — LeiaSR just silently falls back to SbS.

Common convention: a `--no-shim` flag on the engine's build script for fast iteration on the non-LeiaSR codepath, plus an auto-detect that tries `cmake -A x64` and accepts a failure as "no MSVC, skip the shim":

```sh
if [ "$DO_SHIM" = 1 ] && [ -f leiasr_shim/CMakeLists.txt ]; then
    if cmake -S leiasr_shim -B leiasr_shim/build -A x64 >/tmp/shim-cfg.log 2>&1; then
        cmake --build leiasr_shim/build --config Release >/tmp/shim-build.log 2>&1 \
            && echo ">> shim built" \
            || echo ">> shim compile failed (LeiaSR will fall back to SbS)"
    fi
    # On Linux / no-MSVC machines, configure fails fast (no VS generator);
    # we accept that silently rather than treating it as a build error.
fi
```

#### Runtime layout

```
srb2kart.exe                  ← MinGW-built engine, no SR symbols in its imports
SDL2.dll, libcurl-x64.dll ... ← regular engine runtime DLLs
leiasr_shim.dll               ← MSVC-built shim, the only SR-side artifact we ship
<SR runtime DLLs>             ← from system install, found by loader via PATH
```

The SR runtime DLLs (`DimencoWeaving.dll`, `SimulatedRealityCore.dll`, `SimulatedRealityOpenGL.dll`, `opencv_world343.dll`, etc.) live in the system-wide SR Platform install directory (`C:\Program Files\LeiaSR\Platform\bin` on current installers, added to `PATH` by the SR installer; older SDK drops used a `%LEIASR_SDKROOT%\bin` env var). We do **not** ship them — they're shared between every SR-using app on the machine, and shipping our own copy would risk version conflicts. The shim's `/DELAYLOAD` + `LoadLibraryW` preflight handles the case where they aren't on `PATH`: `srk_init` cleanly returns 0, the engine logs a notice, LeiaSR mode silently presents plain SbS.

#### CI considerations

If your engine's CI cross-compiles on Linux (`ubuntu-latest` + `gcc-mingw-w64-x86-64-win32`), the shim can't be built in that same job — MSVC is Windows-only. Standard split:

1. A **separate `windows-latest` job** builds the shim with native MSVC, uploads `leiasr_shim.dll` as an artifact (`actions/upload-artifact@v4`, `if-no-files-found: error`).
2. The **main (Linux MinGW) job** uses `needs: [build-shim]` + `if: always()` so it runs even if the shim job failed, then `actions/download-artifact@v4` with `continue-on-error: true` to optionally pull the shim into `leiasr_shim/build/Release/` before the engine's build/package step picks it up.
3. Mark the shim job `continue-on-error: true` so a missing-secret / private-submodule failure doesn't sink the release.

##### Fetching a private submodule in CI (the PAT recipe)

**`bo3b/SR-lib` is public (A.0), so you do not need any of this for the SDK
itself** — `submodules: recursive` fetches it unauthenticated. This recipe is for
the case where you have mirrored the SDK into a private repo of your own, or
have some other private submodule in the tree. Such a job needs a **PAT with
read access to it**, stored as a repository secret — `SHARED_PAT` is the usual
name if you have one PAT spanning several private repos:

```yaml
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: false        # fetch SR-lib ourselves, after fixing auth

      - name: Fetch SR-lib
        id: srlib
        shell: bash
        env:
          SUBMODULE_PAT: ${{ secrets.SHARED_PAT || secrets.SUBMODULE_PAT }}
        run: |
          if [ -z "$SUBMODULE_PAT" ]; then
            echo "available=false" >> "$GITHUB_OUTPUT"
            echo "::warning::No SHARED_PAT/SUBMODULE_PAT secret; skipping the LeiaSR shim."
            exit 0
          fi

          git config --local --unset-all http.https://github.com/.extraheader || true
          git config --global url."https://x-access-token:${SUBMODULE_PAT}@github.com/".insteadOf "https://github.com/"
          git submodule update --init libs/SR-lib
          git config --global --remove-section url."https://x-access-token:${SUBMODULE_PAT}@github.com/" || true

          echo "available=true" >> "$GITHUB_OUTPUT"

      - name: Build shim
        if: steps.srlib.outputs.available == 'true'
        run: |
          cmake -S leiasr_shim -B leiasr_shim/build -A x64
          cmake --build leiasr_shim/build --config Release
```

Two lines there are load-bearing and both cost real debugging time:

- **Unset `.extraheader` first.** `actions/checkout` writes an
  `http.https://github.com/.extraheader` token into `.git/config` scoped to
  *this* repo, and git applies it to **every** `github.com` URL. Against a
  third-party repo it 401s. It has to come out before your own credentials can
  take effect — leaving it in place makes the PAT look broken.
- **The rewrite must be `--global`.** Not `--local`, and not `git -c`.
  `git submodule update` spawns a **separate `git clone` process** for the
  submodule, and that child reads global/system config plus its own (not-yet-
  existing) repo config — it never sees the parent repo's local config. A
  `--local` `insteadOf` is silently ignored and the clone falls back to
  unauthenticated, which on a private repo means a credential prompt (i.e. a
  hang or failure, not a clear auth error). `git -c` fails for the same reason.

Gate the build step on the secret's presence rather than letting the clone fail,
so a fork or a PR from outside the repo gets a clean warning and a LeiaSR-less
build instead of a red job.

The host engine is unchanged across all three cases. Whether `leiasr_shim.dll` was built locally, downloaded from CI, or never built at all is transparent — it either exists next to the `.exe` or it doesn't, and `LoadLibraryA` decides what happens at runtime.

#### Trade-offs vs. direct MSVC integration

**Pros of the shim:**
- Engine build stays pure-MinGW — no MSVC dependency for the main artifact, no per-TU `/EHa` requirement, no MSVC-specific link flags leaking into the engine's Makefile.
- The shim is ~150 lines of code + a 70-line CMakeLists. Self-contained.
- All the delay-load + SEH-preflight machinery lives in one MSVC translation unit, where MSVC's exception model is in scope.
- If a user doesn't want LeiaSR, they don't need to install MSVC to build the engine — the shim just doesn't get built.

**Cons:**
- One extra build step (a separate `cmake` invocation for the shim) and a small CI split (`windows-latest` job for the shim).
- Extra DLL to ship.
- Marginal `LoadLibrary` + 3× `GetProcAddress` overhead at first use (~microseconds; the SR initialize() call dwarfs it).
- The C ABI is a snapshot — if you add a new SR feature (e.g. eye-position queries for native rendering), you have to add a new `srk_*` export and re-deploy the shim.

The shim build is fast (~3 s incremental on a warm tree). For most edits to the host engine it doesn't rebuild at all. The pattern scales — if you later want a DX backend in the same engine, you add a `dx_weaver_init/weave/shutdown` set of exports to the same shim, no new architecture.

### Hosting the GL weaver inside an `opengl32.dll` proxy

*Added in the nautymac fork, from OGL3D-SR ([`src/stereo.cpp`](https://github.com/nautymac/OGL3D-SR/blob/main/src/stereo.cpp), `GlStateGuard`). The `api-proxy-stereo` skill covers the proxy itself.*

When the weave runs inside a proxy that replaces `opengl32.dll` for a game you don't own, three things differ from an engine that owns its GL state.

**1. The weaver's own GL calls go through your hooks.** The SR GL weaver imports `opengl32.dll`, and in the game's folder that is you.
- Run `weave()` with your per-thread context pointer cleared, so every hooked entry point is a pure pass-through while it runs: no shader injection, no draw duplication, no state tracking. Restore the game's tracked state afterwards.
- Likewise, mark the weaver's creation (`CreateGLWeaver`, where it compiles its shaders) so your shader rewriter leaves those programs alone (`t_ownShaders` on the `flato3d` branch).

**2. The game's frame-end state reaches the weaver's full-screen pass.** A game can end its frame with blending still on. Broken Age ended some shots with `glBlendFunc(GL_DST_COLOR, GL_ONE_MINUS_SRC_ALPHA)`. The woven image was then multiplied into the previous one every frame, and the screen ran to black within frames: "black until I click". An additive leftover made it very bright instead. The eyes themselves were bright the whole time.

The guard around `weave()` (and around your overlay) therefore *sets* state as well as saving it:
- It disables blend, depth test, cull face, scissor, stencil, `GL_FRAMEBUFFER_SRGB`, colour logic op, alpha-to-coverage, rasterizer discard, clip distances 0–2, and alpha test in compatibility contexts.
- It forces `glPolygonMode(GL_FILL)`, all colour mask bits, and a full viewport.
- It restores everything afterwards: per-enable on core contexts, `glPushAttrib(GL_ALL_ATTRIB_BITS)` / `glPushClientAttrib` on compatibility contexts.
- Push and pop with framebuffer 0 bound: `glPopAttrib` restores the draw/read buffers *of the bound framebuffer*.
- Rasterizer discard is in no attrib group; save it yourself.

Logging what each frame ends with ("frame N ends with blend 1 (0x306, 0x303) …", changes only) found this in minutes.

**3. Clip control.** WineD3D (D3D9/10/11 on OpenGL) keeps the D3D convention: `glClipControl(GL_UPPER_LEFT, GL_ZERO_TO_ONE)`. The weaver's and overlay's full-screen passes are written for GL's convention and came out **upside down**. If `GL_CLIP_ORIGIN` / `GL_CLIP_DEPTH_MODE` aren't `GL_LOWER_LEFT` / `GL_NEGATIVE_ONE_TO_ONE`, set them for the weave and put them back afterwards. This is on OGL3D-SR's `flato3d` branch. Before it, the whole woven Huntdown picture was upside down (Door Kickers: Action Squad, only the overlay); with it, Huntdown's title wove upright.

Two related GL notes from the same proxy:
- **Don't present a second GL window** from a DPI-unaware process to get a native-resolution weave. It crashed in NVIDIA's DXGI-based GL present. Share the eyes into D3D11 with `WGL_NV_DX_interop2` and use the DX11 weaver (A.13.4).
- The GL weaver draws into **the bound draw framebuffer at the current viewport**. Bind framebuffer 0 (or your own back-buffer stand-in) and set the viewport to the window just before `weave()`. Never leave the game's last FBO bound.

## B.2 DirectX 12

| Thing | Concrete value |
|---|---|
| Weaver type | `SR::IDX12Weaver1*` |
| Create function | `SR::CreateDX12Weaver(SRContext*, ID3D12Device*, HWND, IDX12Weaver1**) → WeaverErrorCode` |
| Header | `<sr/weaver/dx12weaver.h>` |
| CMake | `SRLib::SR` via A.0's package; longhand is `find_package(srDirectX REQUIRED)` against a full SDK install |
| Link target | `srDirectX::srDirectX` |
| Texture handle type | `ID3D12Resource*` |
| Output destination | the swap-chain RTV that's currently bound — call `setCommandList` / `setViewport` / `setScissorRect` on the weaver before `weave()` |
| Filter setting for upscale | sampler filter is internal to the weaver; nothing to set on the input texture descriptor |
| Weaver destruction | `weaver->destroy()` (IDestroyable pattern). **Don't** `delete weaver` — that's the older deprecated `PredictingDX12Weaver` convention and asserts in debug for `IDX12Weaver1`. |
| Delay-load DLL list | the A.3.1 list with `srDirectX.dll` substituted for `SimulatedRealityOpenGL.dll` |
| Gamma | `setShaderSRGBConversion(read, write)` — decide by what the texture's values **mean**, never by its format's name. Almost every game writes display-encoded (gamma-space) values into a plain `*_UNORM` texture, so the right call is **`(true, true)`**: linearize on read, re-encode on write. `(false, false)` is only right if the values are genuinely linear light. Getting this wrong has **no visible colour symptom** (the round trip is symmetric), only a silent one: the weaver's crosstalk cancellation is calibrated in linear light, so on encoded values it under-cancels in exactly the dark tones where ghosts show, and the anti-crosstalk Strength control looks inert (A.11.1, pitfall #16). Verified 2026-10-19: [dusklight3D](https://github.com/oneup03/dusklight3D) ([aurora3D `lib/webgpu/leiasr.cpp`](https://github.com/oneup03/aurora3D/blob/Stereo3D-rebased/lib/webgpu/leiasr.cpp)) ran `(false, false)` on encoded UNORM content and strength did nothing; `(true, true)` fixed it, matching [VRto3D](https://github.com/oneup03/VRto3D) and [UU3D](https://github.com/oneup03/UU3D), which were already correct. [rt64-3D](https://github.com/oneup03/rt64-3D)/BanjoRecomp3D also uses `(false, false)` on what is probably encoded content, so it most likely under-cancels the same way. That is unverified there. |

### B.2.1 Per-frame weave sequence

```cpp
// 1. Transition the SBS intermediate into SHADER_READ for the weaver to sample.
cmd->resourceBarrier(intermediate, RTV → SHADER_READ);

// 2. Bind the swap-chain RTV — this is where the weave writes.
cmd->setRenderTargets(swapChainRTV);

// 3. ★ CRITICAL: reset the command list's RASTERIZER viewport / scissor to
//    swap-chain dims. The weaver records its draw commands into this command
//    list and the D3D12 rasterizer uses whatever viewport / scissor is
//    currently set on the command list — NOT what we pass to weaver->
//    setViewport() below. If a prior step (e.g. the SBS compose) left the
//    command-list viewport at the wider intermediate size, the weave renders
//    at THAT size and the swap chain shows only the left portion. See
//    B.2.2 for the diagnostic signature.
cmd->RSSetViewports(1, &D3D12_VIEWPORT{0, 0, (float)swapW, (float)swapH, 0, 1});
cmd->RSSetScissorRects(1, &D3D12_RECT{0, 0, swapW, swapH});

// 4. Hand off to the weaver. The (width, height) passed to setInputViewTexture
//    are the FULL combined-SBS texture dimensions (= 2 * perEyeW × perEyeH).
//    setViewport / setScissorRect on the weaver are belt-and-suspenders — the
//    actually-effective viewport is the command list's, set in step 3.
weaver->setInputViewTexture(intermediate, sbsW, sbsH, format);
weaver->setOutputFormat(swapChain.nativeFormat);
weaver->setCommandList(cmd);
weaver->setViewport(D3D12_VIEWPORT{0, 0, (float)swapW, (float)swapH, 0, 1});
weaver->setScissorRect(D3D12_RECT{0, 0, swapW, swapH});
weaver->weave();
```

Two things `weave()` won't tell you about if you get wrong:

1. **Output format mismatch.** The output format **must** match the swap chain's actual `nativeFormat`. Hardcoding `DXGI_FORMAT_R8G8B8A8_UNORM` against a `DXGI_FORMAT_B8G8R8A8_UNORM` swap chain produces visibly broken / over-saturated colors.
2. **Command-list viewport.** The weaver's `setViewport()` is **not** sufficient — D3D12's rasterizer uses whatever `RSSetViewports` last set on the command list. See B.2.2.

### B.2.2 The command-list viewport gotcha

If you do a per-frame compose pass that produces the SBS intermediate at `2W × H` (e.g. via a fullscreen-triangle shader that packs two eye textures side-by-side), and then call into the weaver with the swap-chain RTV bound, **the rasterizer viewport at the moment `weave()` records its draws is still set to the intermediate's `2W × H`** unless you reset it. The weaver then records draws against a `2W × H` rasterizer state, the swap chain is only `W × H` wide, and the user sees:

> "Broken weaved stereo stretched to roughly 2× the display width — only the left half is visible inside the swap chain."

Reproduces 100% on a real Leia panel. The fix is the `cmd->RSSetViewports / RSSetScissorRects` call at step 3 of B.2.1: explicitly bring the command-list viewport back to swap-chain dims before invoking the weaver. The weaver's own `setViewport(viewport)` is not enough on its own — confirmed by stepping through the dimension on actual hardware.

This isn't documented anywhere in the SR SDK headers or the SDK's DX12 sample (which doesn't have a prior compose step to leave the viewport "dirty"). Aurora's working integration ([aurora3D `lib/webgpu/leiasr.cpp`](https://github.com/oneup03/aurora3D/blob/Stereo3D-rebased/lib/webgpu/leiasr.cpp), the `weave()` function) does both: `g_d3dCommandList->RSSetViewports(1, &viewport)` *and* `g_weaver->setViewport(viewport)`. The `RSSetViewports` is the one that actually counts; the weaver call is for the SDK's bookkeeping.

If your host renderer wraps D3D12 in an abstraction (RT64's `RenderCommandList`, Plume's RHI, etc.), the wrapper's `setViewports(...)` call has to fire before the weave. RT64-specific call:

```cpp
commandList->setViewports(RenderViewport(0, 0, (float)swapW, (float)swapH));
commandList->setScissors (RenderRect(0, 0, swapW, swapH));
```

### B.2.3 SBS input dimension convention — confirmed working

For `IDX12Weaver1` (the non-deprecated factory `SR::CreateDX12Weaver`), confirmed against three integrations:

| Reference | Intermediate dims | Width passed to `setInputViewTexture` | Output viewport |
|---|---|---|---|
| Aurora ([aurora3D `lib/webgpu/leiasr.cpp`](https://github.com/oneup03/aurora3D/blob/Stereo3D-rebased/lib/webgpu/leiasr.cpp)) | `2 * perEyeW × perEyeH` | `2 * perEyeW` (the full combined SBS width, = `g_width * 2`) | `perEyeW × perEyeH` |
| SR DX12 sample (`SR-SDK-<version>/x64/examples/directx12_weaving/...`) | `g_viewTextureWidth * 2 × g_viewTextureHeight` | `g_viewTextureWidth` *(per-eye — but this is the sample's naming, see note below)* | `g_windowWidth × g_windowHeight` |
| rt64-3D / BanjoRecomp3D ([`src/render/rt64_leiasr_weaver.cpp`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/render/rt64_leiasr_weaver.cpp)) | `2 * swapW × swapH` | `2 * swapW` (full combined SBS) | `swapW × swapH` |
| **UU3D** ([`src/mods/vr/flat3d/Flat3DCompositorD3D12.cpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DCompositorD3D12.cpp), `weave_leiasr`) | `2 * sbsW × sbsH` | `2 * sbsW` (full combined SBS) | swap-chain dims |

**Pass the FULL combined SBS width on DX12** (= `2 * perEyeW`). Three shipping integrations do (Aurora, rt64-3D/BanjoRecomp3D, UU3D), and it is what the SDK documents: `\param width Width of the texture`, worded identically on the DX9, DX11, DX12 and GL weavers.

**The SR DX12 sample genuinely does pass a per-eye value — do not "correct" your code toward it.** In the sample `g_viewTextureWidth` is `getRecommendedViewsTextureWidth()`, it allocates the texture at `g_viewTextureWidth * 2`, renders each eye into a viewport of that per-eye width, and passes the per-eye width. That is a real divergence from the shipping integrations, not a naming artefact (an earlier revision of this skill claimed it was — it isn't). **Verified-working DX12 = full combined width.**

> ⚠ **Do not generalise the sample's per-eye value to the other backends** — and note the sample is the *only* thing that has ever supported a per-eye reading. An earlier revision of this skill quoted a UU3D comment asserting that per-eye "does not weave correctly on this backend (**unlike the DX11 weaver**)". That parenthetical was never measured, and UU3D's DX11 path was in fact passing per-eye width for a 2W-wide texture — under-reporting it by half — for the entire life of the feature. It was found on 2026-08-10 by reading the SDK docs against the code (every *other* use of the per-eye variable in the same function doubled it) and fixed by routing DX11 through `SRInterfaceDX11`, which reads the dimensions off the resource so the weaver cannot be told a size the texture doesn't have. If you are hand-rolling a bind on any backend, **let the resource desc tell you the width** rather than passing a variable you computed.

> ⚠ **DON'T pass `swapW × swapH` as the input view dims with a `swapW × swapH` intermediate (no doubling anywhere).** That's the "half-SBS" pattern: cheap (one swap-chain-sized intermediate, each half is `W/2 × H`), but the SDK has to upscale each half horizontally before the lenticular interleave — visible quality loss. Use the full-SBS pattern unless the GPU budget really won't fit it.

### B.2.4 Working pattern for engines that already have an SBS compose pass

If the host (RT64, a custom DX12 engine, etc.) already has a stereo compose pass that produces an SBS image from two eye textures, the integration is:

1. **Allocate a `2 * swapW × swapH` intermediate** (`leiaSRComposeTexture` in RT64's case). Each half is a full per-eye view at swap-chain resolution.
2. **Run the engine's SBS compose into the intermediate** with `stereoMode = SideBySide` and `fillFullTarget = true` (or your engine's equivalent), so the entire intermediate is populated — left eye in `[0, swapW)`, right eye in `[swapW, 2*swapW)`.
3. **Composite the UI overlay (if any) INTO the SBS intermediate BEFORE the weave.** Bind the UI texture to both eye slots of the compose (so it appears at zero parallax in each half). Doing the UI overlay *after* the weave makes it show as a 2D-SBS-doubled overlay through the lenticular — visible as two copies of every UI element through the lens.
4. **Transition the intermediate to SHADER_READ, bind the swap-chain RTV.**
5. **Reset the command-list viewport / scissor to swap-chain dims** (B.2.2). This is the step everyone forgets.
6. **Call `weaver->setInputViewTexture(intermediate, 2*swapW, swapH, fmt)`** — the FULL combined SBS width — followed by `setOutputFormat`, `setCommandList`, `setViewport`, `setScissorRect`, and `weave()`.

### B.2.5 RT64 specifics

When the host renderer is RT64 (N64Recomp ports), five details to know:

- **Drive the weave through SR-lib, not the SDK.** [`src/render/rt64_leiasr_weaver.cpp`](https://github.com/oneup03/rt64-3D/blob/Stereo3D/src/render/rt64_leiasr_weaver.cpp) (rt64-3D, `Stereo3D` branch) is a thin adapter over `SimulatedReality::SRInterfaceDX12` (A.0): `CreateSRInterfaceDX12(device, hwnd, &sr)` replaces context creation + weaver creation + `initialize()` in the right order, and `sr->Weave(cmdList, viewport, scissor)` replaces the per-frame setter sequence. This is the configuration running on hardware in Goemon64Recomp3D / BanjoRecomp3D. Two bugs it removes by construction: the context being created with `SRContext::create()` but destroyed with `delete` (the SDK pairs `create()` with `deleteSRContext()`, because the object lives in the SR DLL), and describing the input texture with dimensions passed in by the caller rather than read off the resource desc.
- **`stereoMode = SideBySide`, `fillFullTarget = true`** are the right `StereoRenderer::RenderParams` for the LeiaSR compose. RT64's `StereoComposePS.hlsl` produces the SBS layout under that mode; `fillFullTarget` skips the VI letterbox math that would otherwise pillarbox each eye half.
- **`leiaSRComposeTexture` lives on `PresentQueue`** and gets resized lazily when the swap chain dimensions change. Mirror the same pattern for the `stereoUITexture` used by the in-intermediate UI overlay.
- **`createdGraphicsAPI == D3D12`** is the gate. LeiaSR is DX12-only; on Vulkan / Metal the present-time dispatch must skip the LeiaSR branch and fall through to plain SBS compose. RT64's `External::createdGraphicsAPI` field carries this; check it.
- **Reset the command-list viewport before the weave** — RT64's compose step sets the viewport to the intermediate's `2 * swapW × swapH`. Without an explicit `commandList->setViewports(RenderViewport(0, 0, swapW, swapH))` immediately before the weaver hand-off, the weave renders at 2× width and the swap chain shows only the left half. This is the same gotcha as B.2.2 but it's especially easy to hit in RT64 because the compose pass is right next to the weave.

The compose call ends up looking like:

```cpp
const uint32_t perEyeW = ext.swapChain->getWidth();
const uint32_t perEyeH = ext.swapChain->getHeight();
const uint32_t sbsW    = perEyeW * 2u;
const uint32_t sbsH    = perEyeH;

// ... (lazy-create leiaSRComposeTexture / leiaSRComposeFramebuffer at sbsW × sbsH) ...

StereoRenderer::RenderParams composeParams = stereoParams;
composeParams.stereoMode    = UserConfiguration::StereoMode::SideBySide;
composeParams.targetWidth   = sbsW;
composeParams.targetHeight  = sbsH;
composeParams.fillFullTarget = true;
stereoRenderer->render(composeParams);

// ... (optional UI overlay into the same intermediate) ...

// Transition intermediate to SHADER_READ, bind the swap-chain framebuffer.
commandList->barriers(GRAPHICS, RenderTextureBarrier(leiaSRComposeTexture, SHADER_READ));
commandList->setFramebuffer(swapChainFramebuffer);

// ★ Reset the rasterizer viewport / scissor. Without this, the weave renders
// at sbsW (= 2 * perEyeW) wide and the swap chain only shows the left half.
commandList->setViewports(RenderViewport(0, 0, float(perEyeW), float(perEyeH)));
commandList->setScissors (RenderRect(0, 0, int32_t(perEyeW), int32_t(perEyeH)));

// Hand off to the weaver — full combined SBS width to setInputViewTexture.
leiaSRWeaver.weave(d3dTex, int(sbsW), int(sbsH), DXGI_FORMAT_R8G8B8A8_UNORM,
                   swapChain.nativeFormat, d3dCmd, weaveViewport, weaveScissor);
```

## B.3 DirectX 11

| | |
|---|---|
| Interface | `SimulatedReality::SRInterfaceDX11` (`CreateSRInterfaceDX11(ID3D11DeviceContext*, HWND, &out)`) |
| Underlying weaver | `SR::IDX11Weaver1` via `SR::CreateDX11Weaver` |
| Header | `SR.hpp` — no SDK headers needed in your TU at all |
| Input handle | `ID3D11ShaderResourceView*` over the full **2W × H** SbS texture |
| Input dims | Read off the resource by SR-lib. Do **not** compute them (B.2.3) |
| Output | Whatever RTV is bound when you call `Weave()` |
| Status | Wired against a real integration ([UU3D `Flat3DCompositorD3D11.cpp`](https://github.com/oneup03/UU3D/blob/flat3d-afw/src/mods/vr/flat3d/Flat3DCompositorD3D11.cpp)) and building; **weave not yet confirmed on a panel** |

**The raw-SDK DX11 weaver is confirmed weaving on a panel** (nautymac fork, SR
Platform 1.34.10, Acer SpatialLabs). This uses `SR::CreateDX11Weaver(SRContext*,
ID3D11DeviceContext*, HWND, IDX11Weaver1**)` directly, not SR-lib's facade, so
the UU3D status above is unchanged. Integrations: geo-11_SR SRWeave (weaving
geo-11's SBS back buffer in Unity, DX11 and UE4 games, x64 and x86), OGL3D's
upscale mode (GL eyes shared into D3D11), and the Vulkan → D3D11 interop of VK3D
and Azahar-SR (B.4). Per frame: `setInputViewTexture(srv, eyeW, eyeH, format)`
only when the input changes (A.5), then bind the output RTV, set a full viewport
and scissor, `weave()`, and restore the game's context state.

Simpler than DX12 (B.2): the DX11 weaver holds its own device context, so there
is no per-frame command-list / viewport / scissor handshake. Set the input once,
then bind + `Weave()` each frame.

```cpp
// once
HRESULT hr = SimulatedReality::CreateSRInterfaceDX11(ctx, hwnd, &sr);
if (FAILED(hr) || sr == nullptr) { /* log, fall back to SbS */ }
sr->SetShaderSRGBConversion(true, true);   // for display-encoded content; see the B.2 gamma note

// on SbS (re)creation only — see A.5
sr->SetInputTexture(sbs_srv);              // full 2W x H

// per frame
ctx->OMSetRenderTargets(1, &backbuffer_rtv, nullptr);
ctx->RSSetViewports(1, &swapchain_viewport);
try { sr->Weave(); } catch (...) { /* disable, fall back */ }
```

Differences from DX12 worth knowing:

- **No `SetOutputFormat`.** The DX11 weaver takes the bound RTV as-is.
- **`SetInputTexture` takes an SRV, not the resource.** SR-lib walks
  `SRV → GetResource → ID3D11Texture2D → GetDesc` for the dimensions and takes
  the format from the *view* desc, so a typeless resource is fine as long as the
  SRV is concrete.
- **`CreateSRInterfaceDX11` takes the device context**, so the `setContext` call
  a hand-rolled integration needs is already done.
- The viewport still matters, for the same reason as B.2.2 — just set it on the
  device context instead of a command list.

## B.4 Vulkan (through D3D11 interop)

*Added in the nautymac fork. Verified on an Acer SpatialLabs panel (SR Platform 1.34.10) in two integrations that are not public: **VK3D** (a `vulkan-1.dll` proxy) and **Azahar-SR** (the 3DS emulator's Vulkan renderer, `vk_sr_weaver.cpp` + a bridge DLL).*

The SR SDK 1.34.10 has **no Vulkan weaver**: its `sr/weaver/` headers are GL, DX9, DX10, DX11 and DX12 only. So the Vulkan frame goes to the DX11 weaver through shared resources, and the woven picture comes back into the swapchain image. All synchronisation stays on the GPU.

| | |
|---|---|
| Weaver type | `SR::IDX11Weaver1` (B.3), on your own D3D11 device |
| Create function | `SR::CreateDX11Weaver(SRContext*, ID3D11DeviceContext*, HWND, IDX11Weaver1**)` |
| D3D11 device | Created on the **same adapter** as the Vulkan device: match `VkPhysicalDeviceIDProperties::deviceLUID` (needs `deviceLUIDValid`) against `DXGI_ADAPTER_DESC::AdapterLuid`. Use `ID3D11Multithread::SetMultithreadProtected(TRUE)`, `ID3D11Device5` / `ID3D11DeviceContext4` for fences. |
| Shared textures | Two D3D11 textures, `D3D11_RESOURCE_MISC_SHARED_NTHANDLE \| D3D11_RESOURCE_MISC_SHARED`, NT handles from `IDXGIResource1::CreateSharedHandle`. **Input** = the SBS frame (SRV); **output** = swapchain-sized (RTV). |
| Into Vulkan | `VkExternalMemoryImageCreateInfo{VK_EXTERNAL_MEMORY_HANDLE_TYPE_D3D11_TEXTURE_BIT}` on the image, then `VkImportMemoryWin32HandleInfoKHR` + `VkMemoryDedicatedAllocateInfo` on the allocation, memory type from `vkGetMemoryWin32HandlePropertiesKHR` ∩ the image's requirements. |
| GPU sync | `ID3D11Device5::CreateFence(0, D3D11_FENCE_FLAG_SHARED)` → NT handle → a Vulkan **timeline** semaphore imported with `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_D3D11_FENCE_BIT` (an alias of `…_D3D12_FENCE_BIT`) via `vkImportSemaphoreWin32HandleKHR`. |
| Device extensions | `VK_KHR_external_memory(_win32)`, `VK_KHR_external_semaphore(_win32)` (the non-win32 halves are core in 1.1), plus `get_memory_requirements2` and `dedicated_allocation` before 1.1; the `timelineSemaphore` feature (core in 1.2, `VK_KHR_timeline_semaphore` before). |
| Output destination | the bound RTV (the shared output texture), then copied into the swapchain image on the Vulkan side |

**Enabling features in a device you don't create** (a proxy layer): if the app's
`VkDeviceCreateInfo::pNext` already holds `VkPhysicalDeviceVulkan12Features`, set
`timelineSemaphore = VK_TRUE` **in that struct**. Chaining your own
`VkPhysicalDeviceTimelineSemaphoreFeatures` next to it is invalid. Only append
your own struct when neither is present. Turn on only features the physical
device reports.

Per frame, at `vkQueuePresentKHR` (VK3D) or the renderer's present step
(Azahar). `v` is the timeline's last value, so the steps use `v+1`, `v+2`, `v+3`:

```
Vulkan submit A   waits: the app's present semaphores + timeline ≥ v     (D3D11 and the last copy-back are done)
                  copy the SBS frame into the shared INPUT image
                  barrier: release INPUT to VK_QUEUE_FAMILY_EXTERNAL
                  signals: timeline = v+1
D3D11             ctx4->Wait(fence, v+1)
                  OMSetRenderTargets(OUTPUT rtv); RSSetViewports(full)
                  weaver->setInputViewTexture(inputSrv, eyeW, eyeH, fmt)  (only when it changed, A.5)
                  weaver->weave()            (on failure: CopyResource(OUTPUT, INPUT) so SBS still shows)
                  ctx4->Signal(fence, v+2); Flush()      ← ALWAYS, even when weave() threw
Vulkan submit B   waits: timeline ≥ v+2 (+ the acquire semaphore if you acquire here)
                  barrier: acquire OUTPUT from VK_QUEUE_FAMILY_EXTERNAL
                  copy OUTPUT into the swapchain image → PRESENT_SRC
                  barrier: release OUTPUT to VK_QUEUE_FAMILY_EXTERNAL
                  signals: your present semaphore + timeline = v+3
vkQueuePresentKHR waits on your present semaphore instead of the app's
```

Things that cost time:

- **Signal the fence on every path.** The Vulkan side waits for that value before copying the output. A weave that throws and skips the `Signal` hangs the queue for good.
- **Transfer queue-family ownership** to and from `VK_QUEUE_FAMILY_EXTERNAL` on both shared images, every frame. Keep them in `VK_IMAGE_LAYOUT_GENERAL` while D3D11 owns them; both start out owned by D3D11.
- **Keep a CPU fallback** for devices without timeline or external-semaphore support: a `D3D11_QUERY_EVENT` polled after `Flush`, and the Vulkan submit's own fence. It stalls every frame. VK3D has one, but it was never exercised, because the test GPU always got GPU sync. Log which mode you got ("SR sync: GPU" / "SR sync: CPU").
- **Input width**: both integrations hand `setInputViewTexture` the per-eye width of a `2 × eyeW` input and weave correctly (the A.1 field note).
- **Overlay and cursor go into the SBS input before the hand-off.** The weave consumes the whole picture, so anything drawn afterwards is not woven.

Measured: Azahar (Super Mario 3D Land with a 60 fps patch, 9× internal resolution) reached 60 fps at full emulation speed with ~40 % GPU through this path. The same emulator's OpenGL path with the GL weaver already ran below full emulation speed at 6× and managed 23 fps at 9×. VK3D wove Lara Croft GO (D3D9 → DXVK → Vulkan) at 60 fps with GPU sync. Tested on an RTX 4080 laptop GPU. The interop has not been tried on AMD or Intel.

VKS3D's 3D Vision path (`src/present_nv3d.cpp` in ThreeDeeJay/VKS3D) is a public example of the same hand-off to a D3D consumer (`vulkan-multiview-stereo` §8.2). It uses a D3D12-side fence; the D3D11 fence above works the same way.

## B.5 Direct3D 9

*Added in the nautymac fork, from tridef3d-win11-injector [`srweave/src/dx9.cpp`](https://github.com/nautymac/tridef3d-win11-injector/blob/master/srweave/src/dx9.cpp): weaving TriDef 3D's SBS output in 32-bit D3D9 games. Weaving was confirmed by eye on the panel in Binary Domain. Left 4 Dead reached its first woven frame and survived two in-game resolution changes (`Reset`).*

| | |
|---|---|
| Weaver type | `SR::IDX9Weaver1` |
| Create function | `SR::CreateDX9Weaver(SRContext*, IDirect3DDevice9*, HWND, IDX9Weaver1**)` |
| Header | `<sr/weaver/dx9weaver.h>` |
| Input | `setInputViewTexture(IDirect3DTexture9*, int width, int height, D3DFORMAT, bool isSRGB)`: a `D3DUSAGE_RENDERTARGET`, `D3DPOOL_DEFAULT` texture (fall back to `D3DFMT_A8R8G8B8` if the back buffer's format is refused) |
| Output | the render target bound when `weave()` runs |
| sRGB | `setOutputSRGBWrite(bool)` plus the `isSRGB` input flag |
| Device loss | `invalidateDeviceObjects()` before `IDirect3DDevice9::Reset`, `restoreDeviceObjects()` after a successful one; drop your own `D3DPOOL_DEFAULT` texture and state block too |
| HWND | the swap chain's `hDeviceWindow` (`GetSwapChain(0)` → `GetPresentParameters`), falling back to the foreground window |

Per frame, at `Present` (device or swap chain):

```
GetBackBuffer(0) → StretchRect(backBuffer → sbsTexture surface, D3DTEXF_POINT)
setInputViewTexture(sbsTexture, w / 2, h, fmt, srgb)        // once per texture (A.5)
stateBlock(D3DSBT_ALL)->Capture(); save RT0, depth surface, viewport
SetRenderTarget(0, backBuffer); SetDepthStencilSurface(nullptr); full viewport
ZENABLE / ZWRITEENABLE / SCISSORTESTENABLE / STENCILENABLE = FALSE
BeginScene()            // may fail harmlessly if the game left a scene open: still weave, EndScene only if it began
weaver->weave()
EndScene(); restore RT0, depth surface, viewport; stateBlock->Apply()
original Present
```

Finding the device to hook when a wrapper owns it is covered in A.13.2 (TriDef's temporary device refused `GetSwapChain(0)`; the real device was found by its vtable).

---

## Reference: files / functions you'll touch

**New:**
- `stereo_leiasr.hpp` / `.cpp` (or named to fit your engine's conventions) — the wrapper module from A.6.

**Modified:**
- The host engine's **present path** — dispatch on stereo mode, call `R_LeiaSR_Weave` when LeiaSR is active.
- The host engine's **shutdown path** — call `R_LeiaSR_Shutdown` on exit.
- The host engine's **CMake** — `add_subdirectory` SR-lib + link `SRLib::SR` + `srlib_apply_delayload()` per A.0 (longhand: `find_package` SR + the `/DELAYLOAD` block per A.3.1).
- The host engine's **window-handle accessor** — `I_GetWindowHandle()` or equivalent per A.7.

**Optionally added:**
- A user-facing cvar / menu option to enable LeiaSR mode (e.g. `cv_stereomode STEREO_LEIASR`).
- A startup log line confirming the SR runtime probe result, so support reports can distinguish "user has the runtime" from "user doesn't."

**NOT touched:** the host engine's stereo render path itself. This skill assumes the SbS texture already exists. If it doesn't, that's a separate problem — see `output3d` (the compose layer), `dynamic3d` (the per-eye projection math) and, for RT64-based ports, `rt64-stereo-3d`, or implement the per-eye render loop and off-axis frustum yourself per the engine's conventions.
