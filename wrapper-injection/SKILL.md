---
name: wrapper-injection
description: Get a stereo-3D wrapper DLL into a closed-source Windows game you don't own, and keep it loaded and alive — choosing the hook point (an opengl32.dll proxy for OpenGL games; Direct3D 8-11 translated to OpenGL by WineD3D-for-Windows plus the GL wrapper), making Windows load it before the game's code (why the proxy must be a non-KnownDLL that the exe or a DLL it imports actually imports; a version.dll loader; a forwarding shim with absolute-path forwarders `\\.\GLOBALROOT\SystemRoot\System32\<dll>.dll.<fn>` for games that import no version.dll; IAT patching for static d3d11/dxgi imports bound before your DllMain; LdrRegisterDllNotification for system DLLs that something else maps first; full-path LoadLibrary), the WineD3D-on-Windows traps (the NVIDIA OpenGL driver's DXGI-layered present grabbing a d3d11.dll by name → renamed WineD3D copies; CSMT on/off; the thread-alert hang that breaks Windows 11 24H2+ critical sections; glClipControl upper-left; texture views and immutable storage; contexts deleted and recreated; multi-threaded GL vs your lock), the OpenGL-proxy traps (wglGetProcAddress returning driver pointers for GL 1.1 names; multisampled default framebuffers that refuse blits; your own shaders getting injected; windows created on a thread with no message pump; hotkeys the game also sees; command-line switches Steam does not pass), and the SR-runtime traps that complement leiasr-integration (the 32-bit Virtual PC probe eaten by Mono's vectored handler; a lens left on after a crash; antivirus false positives on loader DLLs). Use it whenever a wrapper is not loaded, loads too late, loads but the game still draws with the system API, or the game hangs or crashes only with the wrapper in place.
---

# wrapper-injection — getting a stereo wrapper into a game you don't own

> *"The wrapper works on my test app. In this game it never loads / loads but nothing changes / the game hangs."*

Every stereo technique in this repo assumes your code sits between the game and the GPU. With a closed-source game that is the hard part: Windows decides which DLL the game gets, other software (Steam's overlay, the GPU driver, the SR runtime, Mono) has its own ideas, and translation layers bring their own threads. This skill collects what it took to make that work for about fifteen games across OpenGL, Direct3D 9 and Direct3D 11, on an NVIDIA laptop with a LeiaSR panel.

*Added in the nautymac fork. Distilled from [OGL3D-SR](https://github.com/nautymac/OGL3D-SR) (`flato3d` branch) and [Flato3D-SR](https://github.com/nautymac/Flato3D-SR) (its `loader/` and installer), verified in the games named below with SR weaving and side-by-side output.*

Related skills: `output3d` (what the wrapper does with the two eyes), `leiasr-integration` (weaver setup, DPI awareness), `flat2d-layered-depth` (depth for 2D games once you are in), `windows-hang-diagnosis` (when the game stops responding).

---

## 1. Pick the hook point

| What the game renders with | Hook point | Notes |
|---|---|---|
| OpenGL | `opengl32.dll` proxy next to the exe | `opengl32.dll` is not a KnownDLL, so the copy in the game folder wins. Forward everything you do not hook to System32's copy. |
| OpenGL selectable (AGS `driver=OGL`, Unity `-force-glcore` if the build has a GL renderer, a game's own `-ogl` switch) | switch the game to OpenGL, then as above | Many Unity builds have no GL renderer ("OpenGL Core was not built"); check before planning on it. |
| Direct3D 8 / 9 / DirectDraw / 10 / 11 | translate to OpenGL with WineD3D-for-Windows (Federico Dossena's builds), then the OpenGL proxy | `wined3d.dll` imports `opengl32.dll` statically, so the GL proxy sees everything. Native D3D wrappers (3Dmigoto/geo-11) are the other route. |
| Direct3D 12, Vulkan | not this route | `vulkan-multiview-stereo` for Vulkan. |
| One finished picture (CPU-composited, emulators, DOSBox) | no geometry to hook | Only AI 2D→3D helps. |

How to tell what a game uses:

- Read the exe's import **and delay-import** tables. Do not trust the first match: a Unity "embedded player" exe can import `opengl32` and still render D3D11. For Unity, the `Version:` line of `<exe>_Data\output_log.txt` or `%USERPROFILE%\AppData\LocalLow\<company>\<game>\Player.log` names the real API.
- Treat `UnityPlayer.dll` and `*-Shipping.exe` (Unreal) as D3D11 unless the log says otherwise.
- If it imports none of them, the game loads the API at run time; look at a running process's modules.

## 2. Make Windows load you before the game's code

### 2.1 The proxy must be something the game imports at start

A DLL in the game folder only takes effect if:

1. its name is **not** in `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs` (those always come from System32), and
2. something loads it by name while the process starts: the exe's import table, or the import table of a DLL the exe imports.

`version.dll` is the classic loader name because SDL2.dll, UnityPlayer.dll and many engines import it. **Check it per game.** Braid, Anniversary Edition imports no `version.dll` anywhere. The loader never ran, the game drew with the system Direct3D 11, and no log appeared at all. "No log at all" is the tell.

Do the check in the installer and print it (`loader start: a version.dll import` / `shim winmm.dll` / `none`):

```text
exe imports version.dll                                → fine
a DLL in the game folder that the exe imports does     → fine (SDL2.dll, UnityPlayer.dll)
neither, but the exe imports a DLL you have a shim for → add the shim (§2.2)
none of those, but a DLL deeper in the folder does     → probably fine (Samorost 2: Adobe AIR\…\Adobe AIR.dll)
nothing                                                → warn: the loader will not load
```

### 2.2 A forwarding shim for games that import no version.dll

Build a DLL named after a non-KnownDLL the exe does import (`winmm.dll` is the usual candidate because of `timeGetTime`; `dsound.dll`, `dinput8.dll`, `dwmapi.dll` are others). It has two jobs:

1. **Behave exactly like the system DLL.** Forward every export by absolute path, so the forwarder cannot resolve to the shim itself:

   ```cpp
   // generated from the system DLL's export table (names keep their ordinals; NONAME exports by ordinal)
   #pragma comment(linker, "/export:timeGetTime=\\\\.\\GLOBALROOT\\SystemRoot\\System32\\winmm.dll.timeGetTime,@141")
   #pragma comment(linker, "/export:__ord2=\\\\.\\GLOBALROOT\\SystemRoot\\System32\\winmm.dll.#2,@2,NONAME")
   ```

   For 32-bit builds use `SysWOW64`. The forwarder string is split at its last `.`, so keep `.dll` in the module part. Verified with a small test exe in both bitnesses: the calls return real values, and the module list shows `\\.\GLOBALROOT\SystemRoot\System32\winmm.dll` loaded next to the shim.
2. **Bring the loader in.** Give the shim a static import of `version.dll`. Windows then resolves it from the application folder, so your loader loads during process start, before any game code, just as a direct import would. Keep the call in a branch that never runs so the import survives optimisation:

   ```cpp
   BOOL WINAPI DllMain(HINSTANCE inst, DWORD reason, LPVOID) {
       if (reason == DLL_PROCESS_ATTACH) DisableThreadLibraryCalls(inst);
       if (reason == 0xF1A703D) GetFileVersionInfoSizeW(L"", nullptr);   // never true; keeps the import
       return TRUE;
   }
   ```

   Do not `LoadLibrary` from `DllMain` instead.

Leave a `winmm.dll` you did not write alone: mod loaders (the Ultimate ASI Loader) use the same names. Put a marker string in your shim so the installer can recognise its own copy, record it in the install manifest, and remove it on uninstall.

### 2.3 What the loader has to do once it runs

- **Static imports are already bound.** By the time any DllMain runs, the exe's `d3d11.dll!D3D11CreateDevice` points at System32. Patch those IAT slots in the game's own modules (the exe and DLLs in its folder, not your wrappers) to stubs that call the game-folder copy. Astral Ascent and Braid AE import `d3d11`/`dxgi` statically.
- **Explicit loads.** Patch the `LoadLibrary*` / `GetModuleHandle*` / `GetProcAddress` imports of the game's modules, so that a request for `d3d9.dll` etc. gets the game-folder copy even when the game passes a full System32 path or `LOAD_LIBRARY_SEARCH_SYSTEM32`.
- **Someone mapped the system DLL first.** Steam's overlay loads `d3d9.dll` early, and a SteamStub-wrapped exe then reaches it through no import you can see (Door Kickers: Action Squad). Register `LdrRegisterDllNotification`, and when a system copy of a redirected DLL maps, bend its creation entry points to stubs that call yours.
- **Do not touch the modules that must see the real system DLL.** Leave `opengl32.dll`, `wined3d.dll` and the loader itself alone.
- **Log it.** Write one line per redirection (`static import d3d11.dll!D3D11CreateDevice -> w3d11.dll`), plus a verbose switch that logs every LoadLibrary. Most "it does nothing" reports are answered by this log.

## 3. Direct3D through WineD3D on Windows

### 3.1 Rename the WineD3D set (NVIDIA)

NVIDIA's OpenGL driver presents through DXGI ("Vulkan/OpenGL present method: layered on DXGI"). Doing that, it loads `d3d11.dll` / `dxgi.dll` by name. If WineD3D's `d3d11.dll` is already loaded under that name, the driver gets WineD3D's copy and the game hangs.

Ship renamed copies instead (`w3d11.dll`, `wxgi.dll`, `w3d10core.dll`, with the import names inside them patched), and point the game at them from the loader (§2.3). The system `d3d11.dll` stays loaded for the driver. D3D9 is not affected.

### 3.2 CSMT

- WineD3D's command-stream thread (CSMT) must stay **on** for Unity and other engines that create D3D11 resources from several threads. With `csmt=0` they crash at start (an access violation in `wined3d.dll`).
- One D3D9 game deadlocked with CSMT **on** at a menu transition: the main thread waited for the CS thread while the driver's present waited for the window thread. Turn it off per game with `HKCU\Software\Wine\AppDefaults\<exe>\Direct3D`, `csmt` (DWORD) = 0.

### 3.3 The thread-alert hang (Windows 11 24H2 and later)

WineD3D's CS thread sleeps in `NtWaitForAlertByThreadId` (with a timeout while queries are pending), and threads that queue work wake it with `NtAlertThreadByThreadId`. An alert that arrives after the timed wait has returned stays **pending**.

Since 24H2, critical-section waits also sleep on thread alerts. The pending alert wakes the CS thread's next contended `EnterCriticalSection` (the process heap lock, a driver lock), ntdll links its wait block a second time, and the thread spins forever in `RtlpWaitOnCriticalSection`. The game stops responding after seconds to minutes, x86 and x64, **with or without your wrapper**.

Fix: patch `wined3d.dll`'s import of `kernel32!GetProcAddress` and answer "not found" for the two functions. WineD3D then falls back to an auto-reset event, where a leftover signal costs one extra loop:

```cpp
static FARPROC WINAPI WineD3DGetProcAddress(HMODULE m, LPCSTR name) {
    if ((ULONG_PTR)name > 0xFFFF &&
        (!strcmp(name, "NtAlertThreadByThreadId") || !strcmp(name, "NtWaitForAlertByThreadId"))) {
        SetLastError(ERROR_PROC_NOT_FOUND);
        return nullptr;
    }
    return GetProcAddress(m, name);
}
// in DllMain of the GL proxy (wined3d imports it, so wined3d's imports are bound and its code has not run yet),
// and again at the first wglCreateContext in case wined3d was loaded later:
PatchModuleImport(GetModuleHandleW(L"wined3d.dll"), "kernel32.dll", "GetProcAddress", (void*)&WineD3DGetProcAddress);
```

Result in The Messenger: 11 hangs in 19 runs before, none in 10 side-by-side and 5 SR runs after. `windows-hang-diagnosis` has the full investigation and a minimal repro.

### 3.4 What WineD3D's OpenGL looks like to your wrapper

- **`glClipControl(GL_UPPER_LEFT, GL_ZERO_TO_ONE)`.** WineD3D keeps D3D's origin. Every full-screen pass of your own (the SR weaver, overlays) comes out upside down unless you set `GL_LOWER_LEFT, GL_NEGATIVE_ONE_TO_ONE` around it and restore it afterwards.
- **Contexts.** WineD3D creates two contexts, deletes them, and works in a new unshared one. GL object names repeat across them, so key per-context caches (generated shaders, programs) by `HGLRC` and purge them on `wglDeleteContext`.
- **Several GL threads.** Keep GL calls outside your global lock. A driver call made while holding it deadlocked against WineD3D's other GL thread.
- **Draw entry points.** WineD3D draws with `glDraw*BaseInstance` and the indirect forms; hook them all.
- **Shaders.** Its fixed-function replacement shaders are GLSL 4.40 with array outputs (`ffp_varying_texcoord[8]`) and interface blocks. Its vertex stage can be two linked shader objects, one of them without `main()` that writes the outputs. Analyse every attached piece, and give up cleanly on a program with a piece you cannot parse.
- **Views and copies.** Shader resource and render target views are `glTextureView`s. Views need **immutable** storage, so any per-eye clone of a viewed texture must be created with `glTexStorage*` (including `glTexStorage2DMultisample` for MSAA targets), or its right-eye view fails and the framebuffer is incomplete (`0x8cd7`). `CopyResource` becomes `glCopyImageSubData`, so mirror it to the clones.
- **Leftover bindings.** WineD3D leaves old render targets bound on texture units the current program does not read. Decide what a draw samples from the sampler uniforms (or `texture[n]` in ARB programs), not from bindings.
- **Read buffer.** WineD3D leaves `GL_NONE` as the read buffer on its draw framebuffers. Set `GL_COLOR_ATTACHMENT0` before reading pixels back.

## 4. OpenGL proxy traps

- **`wglGetProcAddress` for GL 1.1 names.** SDL+glad (AGS 3.6) fetches even `glDrawArrays` through `wglGetProcAddress`, and NVIDIA/AMD return driver pointers. Draws then bypass your exports, and typically the right eye is black. For the GL 1.1 names you implement, return your own exports.
- **Multisampled default framebuffer.** A 4× MSAA window refuses `glBlitFramebuffer` into it. Draw both eyes in as a textured quad with blending off. A dump of your eye buffers looks fine in this case, so check with a real screen capture.
- **Don't inject your own shaders.** Wrap the weaver's and overlay's GL work so your shader rewrite skips it.
- **Assembly programs and fixed function.** Engines built on NVIDIA Cg (`cg.dll`/`cgGL.dll`) use `!!ARBvp1.0` programs, and their log says `shaders injected 0`. Rewrite the ARB vertex program's position output. Fixed-function games need their projection matrix shifted per eye instead.
- **Command-line switches Steam doesn't pass** (Machinarium's `-ogl`). Patch `GetCommandLineA/W` in the game's modules from DllMain, before the CRT builds `argv`.
- **Windows you create.** A window created on WineD3D's CS thread has no message pump, so the game goes "Not responding". `HTTRANSPARENT` only passes the mouse to windows of the same thread. Create it on the game window's thread with a `WH_GETMESSAGE` hook plus a posted message, and use `ShowWindowAsync` / `SWP_ASYNCWINDOWPOS` from other threads.
- **Hotkeys.** Games often ignore Ctrl and react to the F-key part of your shortcut. In a `WH_GETMESSAGE` hook on the game window's thread, turn the key messages (`WM_KEY*`, `WM_SYSKEY*`, keyboard `WM_INPUT`) of your hotkeys into `WM_NULL`.
- **Vsync.** Games that disable it (`wglSwapIntervalEXT(0)`) tear through a woven image; offer a forced swap interval.

## 5. SR-runtime traps (complements `leiasr-integration`)

- **32-bit Unity games hang when weaving starts.** `DimencoWeaving32.dll` runs the Virtual PC detection instruction `0F 3F 07 0B` under its own SEH handler. Mono's vectored exception handler runs first, treats the illegal instruction as fatal and never returns, so the render thread is stuck in `mono.dll`. Install a first vectored handler (`AddVectoredExceptionHandler(1, …)`, at the first SR call so it is ahead of Mono's) that answers exactly that probe: for `EXCEPTION_ILLEGAL_INSTRUCTION` at bytes `0F 3F 07 0B`, set `Ebx = 0xFFFFFFFF`, `Eip += 4`, continue. The 64-bit runtime has no probe.
- **The lens stays on after a crash.** If a game dies with the lens on, the desktop stays woven. Turning the lens off from another program or restarting SRSession does not help; `Restart-Service 'SR Service'` (as administrator) does.
- **A false lead to skip.** The SR runtime imports `TerminateThread`, which makes it an easy suspect for hangs. In the case above it was not the cause, so test with SR off before blaming it.

## 6. Environment

- **Antivirus.** A `version.dll` loader that patches imports looks like malware to cloud heuristics. Windows Defender flagged the x64 loader as `Trojan:Win32/Cloxer`, and a rebuild was flagged again soon after. Offer a per-file Defender exclusion in the installer, and treat "flat, and the loader file is missing" as a quarantine.
- **GPU.** Shader rewrites were verified on NVIDIA; Intel drivers broke them. On hybrid laptops write `GpuPreference=2;` for the exe under `HKCU\Software\Microsoft\DirectX\UserGpuPreferences`.
- **Locked files.** DLLs of a running game cannot be replaced. Check for the process before deploying.

## 7. Checklist for a new game

1. Read imports and delay imports. For Unity, read the log's `Version:` line. Pick the route (§1).
2. Check the loader will start (§2.1), and add the shim if not (§2.2).
3. First run, then read the loader log: are the static imports redirected, and is WineD3D in use?
4. Read the wrapper log: is it loaded, are contexts created, and do frame statistics advance?
5. If it hangs, follow `windows-hang-diagnosis` before changing anything. Is it the alert hang (§3.3), CSMT (§3.2), or the renamed set (§3.1)?
6. Back up saves before automated runs, and restore files byte for byte afterwards (copy the backup back; do not rewrite values).
