# 3D-Skills

Agent skills for adding stereoscopic 3D output to games, renderers, and mods. Each skill is a `SKILL.md` written for coding agents that support the [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills) format (Claude Code and compatible tools): the frontmatter `description` tells the agent when to load it, and the body is a distilled, engine-agnostic playbook with worked code, the traps that cost real debugging time, and links to public reference implementations.

They were written while shipping stereo 3D in the projects listed under [Reference implementations](#reference-implementations), and every claim in them was checked against code that runs on real displays (active-glasses panels, ultrawide full-SbS, anaglyph, and LeiaSR autostereoscopic panels).

## Skills

| Skill | Use it when you need to… |
|---|---|
| [`output3d`](output3d/SKILL.md) | Turn two per-eye textures into a display-ready stereo image: Side-by-Side, Top-and-Bottom, row/column interlaced, checkerboard, external interop (Katanga-style), LeiaSR, and Red-Cyan Dubois anaglyph. One shared compose shader, per-eye texture cache (AFR-safe), upscale-then-pattern rule, DPI awareness, automatic full-SbS on ~32:9 ultrawide, HDR-aware anaglyph, ghost/crosstalk reduction, and the anaglyph+UI black-screen bug. |
| [`dynamic3d`](dynamic3d/SKILL.md) | Add the adaptive layer on top of an off-axis stereo renderer: the clip-space separation parameterization (NVIDIA/3Dmigoto convention), migration from metres-based separation, depth-buffer-driven auto-convergence with camera-cut snapping, robust depth sampling, and a depth-aware crosshair/HUD. |
| [`extraoutput3d`](extraoutput3d/SKILL.md) | The output paths `output3d` deliberately defers: HDMI 1.4 frame-packed (including programming the display timing), dual-monitor dual-display, and the seven opt-in anaglyph variants beyond Red-Cyan Dubois. |
| [`leiasr-integration`](leiasr-integration/SKILL.md) | Hand a side-by-side image to the LeiaSR (Simulated Reality) weaver: SDK vendoring via `bo3b/SR-lib`, init ordering, delay-load + SEH preflight so the binary launches without the SR runtime, DPI-awareness, the C-ABI shim pattern for MinGW hosts, and per-backend (OpenGL / DirectX 12 / DirectX 11) specifics. |
| [`skybox-stereo-depth`](skybox-stereo-depth/SKILL.md) | Put sky, stars, sun/moon and distant flares at the right stereo depth under clip-space separation: why the depth → ∞ limit is just ±separation × viewWidth/2 (no convergence, FoV, aspect or stage-scale term survives), which of the two remaining strategies to use per element type, the pitfalls (coordinate wrap, double shift, legacy FoV helpers, sign conventions), calibration in screen-width fractions, and the exact conversion from a legacy world-units IPD calibration. |
| [`rt64-stereo-3d`](rt64-stereo-3d/SKILL.md) | Add all of the above to an RT64-based N64 recompilation port: RT64's two-queue architecture, the two-pass per-eye render with override targets, projection classification via transform IDs and the recompilation-side patches that tag them, the atomic config bridge, and the RT64 aspect-rescale trap. |

The skills cross-reference each other by name; every name they mention is in this repo.

## Installing

Each skill is a folder containing a single `SKILL.md`. Copy the folders you want into your agent's skills directory.

Claude Code, user-wide:

```sh
git clone https://github.com/oneup03/3D-Skills.git
cp -r 3D-Skills/output3d 3D-Skills/dynamic3d 3D-Skills/extraoutput3d 3D-Skills/leiasr-integration 3D-Skills/skybox-stereo-depth 3D-Skills/rt64-stereo-3d ~/.claude/skills/
```

Claude Code, per project: copy the same folders into `.claude/skills/` inside the project instead.

Once installed, the agent loads a skill automatically when a task matches its description, or you can invoke one directly with `/output3d`, `/dynamic3d`, `/extraoutput3d`, `/leiasr-integration`, `/skybox-stereo-depth`, or `/rt64-stereo-3d`.

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

## License

MIT — see [LICENSE](LICENSE).
