---
name: windows-hang-diagnosis
description: Find out why a Windows game stops responding ("Not responding", intermittent, after seconds to minutes) when a stereo wrapper, a translation layer (WineD3D, DXVK) or an injected runtime (SR weaver, overlay) is in the process — without a debugger attached while it happens. Covers reproducing an intermittent hang in a loop and recording everything while the process is still hung (every thread's stack, threads suspended by someone else, held critical sections in module images and in RtlCriticalSectionList, and the registers of threads waiting on a lock, which name the lock even when it lives on the heap), a watchdog inside the wrapper that logs the render thread's stack without touching the heap, naming frames with PDBs fetched from the Microsoft symbol server (CodeView RSDS GUID+age) and MinGW/MSVC map symbols, decoding RTL_CRITICAL_SECTION.LockCount, disassembling ntdll to see what a spinning thread is doing, the Windows 11 24H2+ critical-section wait-block design (PEB->WaitOnAddressHashTable, wait blocks on the waiter's stack) and its fatal reaction to a stray NtAlertThreadByThreadId (signature: a loop at RtlpWaitOnCriticalSection with [rbx+0x10] == rbx), a minimal repro of that, how to narrow the cause by removing one component at a time, why stress tests that do not reproduce it still matter, and a table of hang signatures already solved. Use it before changing any code when a game hangs only with your wrapper or translation layer installed, or hangs on newer Windows builds only.
---

# windows-hang-diagnosis — why did the game stop responding?

> *"With the wrapper the game freezes after a minute or two, not every run. Without it, it also froze once."*

A hang in a game you do not own usually involves three or four parties: the game's engine, a translation layer, the GPU driver, your wrapper, and maybe an overlay or an SR runtime. Guessing which one is at fault costs days. The method below took one such hang (The Messenger: Unity → D3D11 → WineD3D → OpenGL → wrapper → SR weaver) from "freezes sometimes" to a one-line fix and a 60-line repro in an evening. The facts about ntdll are from Windows 11 build 26200 and the code from Wine 11.0. Re-check offsets on other builds.

*Added in the nautymac fork. The tools are in [Flato3D-SR `tools/hang`](https://github.com/nautymac/Flato3D-SR/tree/main/tools/hang), including the procedure and the case study.*

Related skills: `wrapper-injection` (the loading and WineD3D traps that cause many of these hangs).

---

## 1. Reproduce it, and record while it is hung

Hangs that come and go need a loop that runs the game, drives it into play, waits, and stops at the first run that hangs. While the process is still hung, it records:

| Record | How | Why |
|---|---|---|
| Threads suspended by someone else | `SuspendThread` on each thread returns the previous suspend count; read the IP, then resume. Do this first. | A GC's stop-the-world, a debugger or a watchdog can freeze a thread halfway through releasing a lock |
| Every thread's stack | An external tool, built for the target's bitness, that suspends each thread and scans its stack for values inside executable image ranges | Heuristic: stale return addresses from earlier calls appear too. Trust the top (`ip`) and the next few frames most |
| Held critical sections in module images | Scan a module's writable sections for `RTL_CRITICAL_SECTION` shapes with `RecursionCount >= 1`, `LockCount != -1` and an `OwningThread` that is a live thread of the process | Static locks such as a translation layer's global mutex (`wined3d_cs`) |
| Held and named locks process-wide (x64) | Walk `ntdll!RtlCriticalSectionList` (address from ntdll's PDB). Wine stores a lock's name in `DebugInfo->Spare[0]` (`"wined3d_device_gl.allocator_cs"`) | Only locks created with a debug record are listed; Wine creates them all with one |
| Registers of waiting threads (x64) | For each thread whose IP is inside `RtlpWaitOnCriticalSection` (range from the PDB), dump the registers and check which ones point at a plausible critical section | The lock a thread waits on is in a register even when it lives on the heap and has no debug record |
| The wrapper's own log | Copy it next to the records | |

How a run counts as hung: either the wrapper's watchdog writes its line (§2), or `Process.Responding` is false three times in a row. The second test works with the wrapper removed, which you will need (§5).

## 2. A watchdog inside the wrapper

A thread that sleeps for a second at a time. If the frame counter has not moved for 4 s, it does this up to three times:

```cpp
SuspendThread(render);                       // it may hold the heap, loader or log lock right now:
GetThreadContext(render, &ctx);              // only copy raw values while it is suspended -
memcpy(words, (void*)ctx.Rsp, n);            // a static buffer, no heap, no logging
ResumeThread(render);
// then, with the thread running again: describe ctx.Rip and every word that points into an image,
// via VirtualQuery + GetMappedFileName (no loader lock), and log "HANG? no frame for 4 s (frame N): ..."
```

Logging from inside the suspended window (allocating, formatting, `GetModuleHandleEx`) deadlocks the watchdog on the very lock the render thread holds.

## 3. Name the frames

- **System DLLs:** read the CodeView record (`RSDS`, GUID, age, PDB name) from the DLL's debug directory and download `https://msdl.microsoft.com/download/symbols/<pdb>/<GUID without dashes, upper case><age in hex>/<pdb>`. Load it with the system `dbghelp.dll` (`SymLoadModuleEx` on the local PDB). A dbghelp copy from a Visual Studio diagnostics folder failed to load (error 126). Fetch again after every Windows update.
- **Your wrapper:** have the build emit a linker `.map` file and look addresses up in it.
- **MinGW builds:** some keep a COFF symbol table, others are stripped. WineD3D 11.0 from fdossena keeps one; 11.19 is stripped. Reproduce the same hang with a build that has symbols when the names matter.
- **Everything else:** use the nearest export. A large offset means an internal function, so use the name only as a hint.

## 4. Read what you recorded

### 4.1 `RTL_CRITICAL_SECTION.LockCount` (Windows 8 and later)

| Value | Meaning |
|---|---|
| `-1` | free |
| bit 0 clear (even) | held |
| bit 1 clear | a waiting thread has been woken and is expected to take the lock |
| `(-1 - LockCount) >> 2` | number of waiting threads |

### 4.2 Waiting in the kernel or spinning?

- **IP at an `Nt…` system-call stub** (`ntdll+0x16xxxx` on this build): the thread is waiting in the kernel.
- **IP that changes between samples inside `RtlpWaitOnCriticalSection`:** the thread is spinning in user mode.
- **IP that keeps the same address in a tight loop:** disassemble it (`dumpbin /disasm:nobytes /range:<start>,<end> ntdll.dll`, image base from `dumpbin /headers`) and read the loop. That is how the case below was solved.

### 4.3 Interpretation table

| What you see | Meaning |
|---|---|
| A waits for B's lock, B waits for A's | Lock-order deadlock |
| One thread `Sleep`s/spins waiting for another while holding a lock that the other one needs | Waiting while holding a lock (a translation layer's global mutex is the usual one) |
| The owner of the awaited lock is a thread that no longer exists | The owner ended or was killed (`TerminateThread`) while holding it |
| The lock is free (`-1`, or `-3` = free + woken mark) and the waiter still spins | A lost or a stray wake-up (§6). Suspect suspended threads and thread alerts |
| A suspended thread stopped in heap or lock code | Stopped by a GC/debugger halfway through a release |

## 5. Narrow it down — one change at a time

Run the same number of times (5 or more) after each change. Back up every file you swap and restore it **by copying the backup back**: rewriting a value drops things like a BOM.

| Change | Tells you |
|---|---|
| Remove the wrapper (rename the proxy DLL) | Whether the wrapper is needed for the hang at all |
| Output without the SR weaver (side-by-side) | Whether the SR runtime is involved |
| Translation-layer threading (`csmt=0` for WineD3D) | Whether its worker thread is involved (Unity crashes without it) |
| Another translation-layer version | Regression or not, and a build with symbols |
| Driver settings: NVIDIA OpenGL threaded optimisation (DRS id `0x20C1221E`: 1 on, 2 off), present method (`0x20D690F8`: 0 native, 1 layered on DXGI) | Whether the driver's own threads or present path take part |

**Stress tests are worth writing even when they fail to reproduce.** Write a small program that hammers the suspected mechanism: suspend/resume storms on critical-section waiters, `WaitOnAddress` traffic on addresses that share a hash bucket, heap traffic, short-lived threads. If millions of operations never hang, Windows is not broken in general, and the trigger is something unusual inside *this* process. Then search every DLL in the process for the names of low-level functions, as **strings** and not only as imports, because a translation layer may fetch them with `GetProcAddress`.

## 6. The Windows 11 24H2+ critical-section trap

### 6.1 What the code does (build 26200)

- A contended `EnterCriticalSection` ends up in `RtlpWaitOnCriticalSection`.
- The waiter builds a wait block on its own stack: `+0` the address waited on (`&LockCount`), `+8` its thread id, `+0x10` next, `+0x18` prev, `+0x20` tail, `+0x28` state.
- It pushes the block onto one of 128 lists at `PEB+0x3a0` (`WaitOnAddressHashTable`, bucket `(addr >> 5) & 0x7f`), shared with `WaitOnAddress` waits.
- It sleeps in `NtWaitForAlertByThreadId`.
- After waking, it does **not** check that its block was actually unlinked. If the lock still looks contended, it pushes the same block again.

### 6.2 The failure

The thread was woken by an alert that did not come from the lock's releaser: a `NtAlertThreadByThreadId` that had been sent earlier and stayed pending. So it pushes a block whose address is already the list head. Its `next` becomes itself, and the walk that looks for the list's tail never ends:

```text
RtlpWaitOnCriticalSection+0x2d0:  mov rdx,rbx / mov rbx,[rbx+10h] / mov [rbx+18h],rdx
                                  cmp qword ptr [rbx+20h],0 / jne … / jmp +0x2d0      ; [rbx+0x10] == rbx forever
```

**Signature:** the IP stays at `RtlpWaitOnCriticalSection+0x2db…0x2e6` (`ntdll+0xfc9b` on x64, `ntdll+0x6ad38` in WOW64), `rbx` points into the thread's own stack, and `[rbx+0x10] == rbx`. The lock itself may be free.

### 6.3 Who sends stray alerts

Anything that calls `NtAlertThreadByThreadId` for a thread that is not waiting for it. WineD3D-for-Windows does exactly that: its command-stream thread waits for work with `NtWaitForAlertByThreadId` and a timeout, and producers alert it whenever its `waiting_for_event` flag is set (`dlls/wined3d/cs.c`). `wrapper-injection` §3.3 has the fix.

### 6.4 Minimal repro (hangs on 24H2+, x86 and x64)

```cpp
// a pending alert, then a contended EnterCriticalSection: the waiter never comes back
using AlertFn = LONG(WINAPI*)(HANDLE);
CRITICAL_SECTION cs; HANDLE ready, go;
DWORD WINAPI Waiter(void*) { SetEvent(ready); WaitForSingleObject(go, INFINITE);
                             EnterCriticalSection(&cs); LeaveCriticalSection(&cs); return 0; }
DWORD WINAPI Holder(void*) { EnterCriticalSection(&cs); Sleep(500); LeaveCriticalSection(&cs); return 0; }
int main() {
    auto alert = (AlertFn)GetProcAddress(GetModuleHandleA("ntdll.dll"), "NtAlertThreadByThreadId");
    InitializeCriticalSectionAndSpinCount(&cs, 0);
    ready = CreateEventW(0, TRUE, FALSE, 0); go = CreateEventW(0, TRUE, FALSE, 0);
    DWORD tid; HANDLE w = CreateThread(0, 0, Waiter, 0, 0, &tid);
    WaitForSingleObject(ready, INFINITE);
    CreateThread(0, 0, Holder, 0, 0, 0); Sleep(50);       // the holder owns the lock
    alert((HANDLE)(ULONG_PTR)tid);                         // stray alert: stays pending
    SetEvent(go);
    return WaitForSingleObject(w, 5000) == WAIT_OBJECT_0 ? 0 : 3;   // 3 = stuck
}
```

## 7. Hang signatures already solved

| Signature | Cause | Where |
|---|---|---|
| Loop at `RtlpWaitOnCriticalSection+0x2db`, `[rbx+0x10] == rbx`, usually WineD3D's CS thread inside a heap or driver lock | Stray thread alert (§6) | `wrapper-injection` §3.3 |
| Unity graphics thread in `wined3d_device_context_emit_map` → `wined3d_cs_mt_finish` (Sleep loop) holding `wined3d_cs` | The visible half of the above: it waits for the stuck CS thread | same |
| D3D11 game under WineD3D hangs at start on NVIDIA | The driver loaded WineD3D's `d3d11.dll` by name | `wrapper-injection` §3.1 |
| D3D9 game under WineD3D hangs at a menu transition | CSMT plus the driver's DXGI present waiting on the window thread | `csmt=0` for that exe |
| 32-bit Unity game hangs exactly when SR weaving starts, render thread in `mono.dll` | The SR runtime's Virtual PC probe eaten by Mono's vectored handler | `wrapper-injection` §5 |
| "Not responding" and dead clicks once your presentation window exists | Window created on a thread without a message pump | `wrapper-injection` §4 |
| An 80-100 s stall after frame 3 on some first runs, then it continues | Unknown; it is not a hang | Retry in automation |

## 8. Wrong leads (so you can skip them)

- **The SR runtime imports `TerminateThread`.** One hang happened right as SR started, but the same hang happened with SR off.
- **Mono's GC suspending threads.** No thread was suspended at the moment of the hang, and suspend/resume storms never reproduced it.
- **Plain contention on the heap lock.** Stress tests with millions of acquisitions never hung.
- **Screen captures as evidence.** GDI captures of full-screen OpenGL/WineD3D windows are black. Use the wrapper's eye-buffer dump or a person looking at the screen.
