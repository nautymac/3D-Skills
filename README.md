# 3D-Skills

Agent skills for adding stereoscopic 3D output to games, renderers, and mods. Each skill is a `SKILL.md` written for coding agents that support the [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills) format (Claude Code and compatible tools): the frontmatter `description` tells the agent when to load it, and the body is a distilled, engine-agnostic playbook with worked code, the traps that cost real debugging time, and links to public reference implementations.

They were written while shipping stereo 3D in the projects listed under [Reference implementations](#reference-implementations), and every claim in them was checked against code that runs on real displays (active-glasses panels, ultrawide full-SbS, anaglyph, and LeiaSR autostereoscopic panels).

## Skills

| Skill | Use it when you need to… |
|---|---|
| [`output3d`](output3d/SKILL.md) | Turn two per-eye textures into a display-ready stereo image: Side-by-Side, Top-and-Bottom, row/column interlaced, checkerboard, external interop (Katanga-style), LeiaSR, and Red-Cyan Dubois anaglyph. One shared compose shader, per-eye texture cache (AFR-safe), upscale-then-pattern rule, DPI awareness, automatic full-SbS on ~32:9 ultrawide, HDR-aware anaglyph, ghost/crosstalk reduction, and the anaglyph+UI black-screen bug. |
| [`dynamic3d`](dynamic3d/SKILL.md) | Add the adaptive layer on top of an off-axis stereo renderer: the clip-space separation parameterization (NVIDIA/3Dmigoto convention), migration from metres-based separation, depth-buffer-driven auto-convergence with camera-cut snapping, robust depth sampling, and a depth-aware crosshair/HUD. |
| [`extraoutput3d`](extraoutput3d/SKILL.md) | The output paths `output3d` deliberately defers: HDMI 1.4 frame-packed (including programming the display timing), dual-monitor dual-display, and the seven opt-in anaglyph variants beyond Red-Cyan Dubois. |
| [`leiasr-integration`](leiasr-integration/SKILL.md) | Hand a side-by-side image to the LeiaSR (Simulated Reality) weaver: SDK vendoring via `bo3b/SR-lib`, init ordering, delay-load + SEH preflight so the binary launches without the SR runtime, DPI-awareness, the C-ABI shim pattern for MinGW hosts, tuning the weaver's own anti-crosstalk (and the sRGB-flag trap that silently disables it), and per-backend (OpenGL / DirectX 12 / DirectX 11) specifics. |
| [`flat2d-layered-depth`](flat2d-layered-depth/SKILL.md) | Give depth to flat 2D games (pixel art, hand-drawn, comic, Flash-like) whose draws carry none: sort the game by kind, a generated geometry shader that shifts each sprite per eye without read-back, the `feet` and `order` depth rules, keeping UI at the screen, following the composition (layer caches, composite layout, views and copies), the exclusions that wreck it otherwise (LUT strips, scratch targets, Unity 2D lit sprites), measuring with dense disparity maps, and per-engine starting settings. *Added in the nautymac fork; from OGL3D-SR / Flato3D-SR.* |
| [`wrapper-injection`](wrapper-injection/SKILL.md) | Get a stereo wrapper into a closed Windows game and keep it alive: the hook point per API, why the proxy must be a non-KnownDLL the game imports at start, a `version.dll` loader plus a forwarding shim (absolute-path forwarders) for games that import no `version.dll`, IAT patching for static `d3d11`/`dxgi` imports, Direct3D through WineD3D-for-Windows (renamed set for NVIDIA, CSMT, the Windows 11 24H2 thread-alert hang, glClipControl, views and contexts), OpenGL-proxy traps, and SR-runtime traps (the 32-bit Virtual PC probe vs Mono). *Added in the nautymac fork.* |
| [`windows-hang-diagnosis`](windows-hang-diagnosis/SKILL.md) | Find why a game with a wrapper or translation layer stops responding: reproduce in a loop and record while hung (stacks, suspended threads, lock owners from registers), an in-wrapper watchdog that does not touch the heap, symbols from the Microsoft symbol server, decoding `LockCount`, narrowing one component at a time, the Windows 11 24H2+ critical-section wait-block trap with a minimal repro, and a table of solved hang signatures. *Added in the nautymac fork.* |
| [`skybox-stereo-depth`](skybox-stereo-depth/SKILL.md) | Put sky, stars, sun/moon and distant flares at the right stereo depth under clip-space separation: why the depth → ∞ limit is just ±separation × viewWidth/2 (no convergence, FoV, aspect or stage-scale term survives), which of the two remaining strategies to use per element type, the pitfalls (coordinate wrap, double shift, legacy FoV helpers, sign conventions), calibration in screen-width fractions, and the exact conversion from a legacy world-units IPD calibration. |
| [`rt64-stereo-3d`](rt64-stereo-3d/SKILL.md) | Add all of the above to an RT64-based N64 recompilation port: RT64's two-queue architecture, the two-pass per-eye render with override targets, projection classification via transform IDs and the recompilation-side patches that tag them, the atomic config bridge, and the RT64 aspect-rescale trap. |
| [`vulkan-multiview-stereo`](vulkan-multiview-stereo/SKILL.md) | Make a Vulkan renderer (or a Vulkan app you don't own — DXVK, Zink, emulator backends) render both eyes in one pass with `VK_KHR_multiview`: 2-layer render targets and multiview render-pass / dynamic-rendering plumbing, SPIR-V patching with `ViewIndex`, UI/sky/fullscreen classification, the sky and mono-UI depth fixes, deferred/post-process fragment fixes, the operations multiview does not cover (copies, barriers, compute, ray tracing), and the traps mined from VKS3D's history. *Added in the nautymac fork; distilled from reading VKS3D's code and history, not yet verified on a display like the skills above.* |

The skills cross-reference each other by name; every name they mention is in this repo.

## Installing

Each skill is a folder containing a single `SKILL.md`. Copy the folders you want into your agent's skills directory.

Claude Code, user-wide:

```sh
git clone https://github.com/oneup03/3D-Skills.git
cp -r 3D-Skills/output3d 3D-Skills/dynamic3d 3D-Skills/extraoutput3d 3D-Skills/leiasr-integration 3D-Skills/skybox-stereo-depth 3D-Skills/rt64-stereo-3d 3D-Skills/vulkan-multiview-stereo 3D-Skills/flat2d-layered-depth 3D-Skills/wrapper-injection 3D-Skills/windows-hang-diagnosis ~/.claude/skills/
```

Claude Code, per project: copy the same folders into `.claude/skills/` inside the project instead.

Once installed, the agent loads a skill automatically when a task matches its description, or you can invoke one directly with `/output3d`, `/dynamic3d`, `/extraoutput3d`, `/leiasr-integration`, `/skybox-stereo-depth`, `/rt64-stereo-3d`, `/vulkan-multiview-stereo`, `/flat2d-layered-depth`, `/wrapper-injection`, or `/windows-hang-diagnosis`.

## Reference implementations

The skills link to these public repositories for complete, working code:

| Project | What it is | Relevant to |
|---|---|---|
| [UU3D](https://github.com/oneup03/UU3D) | Unreal Universal 3D Mod (a UEVR fork) — all output modes, auto-convergence, frame-packed timing, D3D11 + D3D12 LeiaSR | `output3d`, `dynamic3d`, `extraoutput3d`, `leiasr-integration` |
| [rt64-3D](https://github.com/oneup03/rt64-3D) (`Stereo3D` branch; per-game `Goemon3D`, `DK64-3D`, `Dino3D`) | Stereo RT64 renderer used by N64 recompilation ports such as [Goemon64Recomp3D](https://github.com/oneup03/Goemon64Recomp3D), [BanjoRecomp3D](https://github.com/oneup03/BanjoRecomp3D), [Donkey-Kong-64-3D](https://github.com/oneup03/Donkey-Kong-64-3D) and [dino-recomp-3D](https://github.com/oneup03/dino-recomp-3D) | `rt64-stereo-3d`, `output3d`, `dynamic3d`, `leiasr-integration` |
| [VRto3D](https://github.com/oneup03/VRto3D) | OpenVR driver that renders SbS/TaB/anaglyph/LeiaSR | `output3d`, `extraoutput3d`, `leiasr-integration` |
| [RingRacers-3D](https://github.com/oneup03/RingRacers-3D) | OpenGL + MSVC LeiaSR integration | `leiasr-integration` |
| [Kart-Public-3D](https://github.com/oneup03/Kart-Public-3D), [perfect_dark_3D](https://github.com/oneup03/perfect_dark_3D) | MinGW hosts using the C-ABI LeiaSR shim DLL; perfect_dark_3D is also the worked example for sky/star stereo depth | `leiasr-integration`, `skybox-stereo-depth` |
| [aurora3D](https://github.com/oneup03/aurora3D) (`Stereo3D-rebased` branch) | WebGPU-on-D3D12 LeiaSR weave and the DPI-unaware-resize fix, used by [dusklight3D](https://github.com/oneup03/dusklight3D) | `leiasr-integration` |
| [bo3b/SR-lib](https://github.com/bo3b/SR-lib) (`api_expansion` branch) | LeiaSR SDK packaged as a git submodule with a CMake package and `SRInterface*` wrapper | `leiasr-integration` |
| [ThreeDeeJay/VKS3D](https://github.com/ThreeDeeJay/VKS3D) (GPL-3.0; studied, not copied) | Drop-in Vulkan ICD that injects multiview stereo into arbitrary Vulkan apps by patching SPIR-V, with generic sky/UI depth fixes and SbS / 3D Vision output | `vulkan-multiview-stereo` |
| [OGL3D-SR](https://github.com/nautymac/OGL3D-SR) (`flato3d` branch) | `opengl32.dll` stereo proxy with SR weaving / SbS / TaB; the generated flat-depth geometry shader, ARB-program rewrite, WineD3D handling and hang watchdog | `flat2d-layered-depth`, `wrapper-injection`, `windows-hang-diagnosis`, `leiasr-integration` |
| [Flato3D-SR](https://github.com/nautymac/Flato3D-SR) | Installer and profiles that make flat 2D games stereo through OGL3D-SR (D3D games via WineD3D); `loader/` (version.dll loader and forwarding shim), `tools/hang/` (hang diagnosis kit), `docs/WORKFLOW.md` | `flat2d-layered-depth`, `wrapper-injection`, `windows-hang-diagnosis` |

## License

MIT — see [LICENSE](LICENSE).
