# Building Roblox Externals — A Practical Guide

A structured walkthrough for Windows developers who want to understand how Roblox externals are built, why they break, and how to design one that survives updates.

This guide is written from real project experience (FF0L / servy-ui). It focuses on architecture, tooling, and debugging — not copy-paste cheat code.

---

## Table of contents

1. [What you are building](#1-what-you-are-building)
2. [Before you start](#2-before-you-start)
3. [How Roblox is laid out in memory](#3-how-roblox-is-laid-out-in-memory)
4. [The external pipeline](#4-the-external-pipeline)
5. [Process attach and memory reads](#5-process-attach-and-memory-reads)
6. [Offsets — the moving target](#6-offsets--the-moving-target)
7. [The LIVE channel trap](#7-the-live-channel-trap)
8. [Resolving the game world](#8-resolving-the-game-world)
9. [Walking the instance tree](#9-walking-the-instance-tree)
10. [Players, characters, and bones](#10-players-characters-and-bones)
11. [Camera, view matrix, and world-to-screen](#11-camera-view-matrix-and-world-to-screen)
12. [ESP — drawing what you read](#12-esp--drawing-what-you-read)
13. [Aim and input (high level)](#13-aim-and-input-high-level)
14. [Overlay UI architecture](#14-overlay-ui-architecture)
15. [Click-through and hit-testing](#15-click-through-and-hit-testing)
16. [Threading and frame budget](#16-threading-and-frame-budget)
17. [Project layout that scales](#17-project-layout-that-scales)
18. [Debugging when nothing works](#18-debugging-when-nothing-works)
19. [Update survival checklist](#19-update-survival-checklist)
20. [Tools worth learning](#20-tools-worth-learning)
21. [Legal and safety notes](#21-legal-and-safety-notes)
22. [Further reading](#22-further-reading)

---

## 1. What you are building

A **Roblox external** is a separate Windows program that:

1. Finds the running Roblox client (`RobloxPlayerBeta.exe`)
2. Opens the process with read (and optionally write) access
3. Uses **version-specific offsets** to locate engine structures in memory
4. Reads game state (players, parts, camera, health, etc.)
5. Presents features through an **overlay** (ESP boxes, menu, watermark) or by sending input

It does **not** inject into Roblox. It runs outside the game. That is why it is called *external*.

```
┌─────────────────────┐         ReadProcessMemory / WriteProcessMemory
│   Your external     │ ───────────────────────────────────────────────►
│   (ff0l.exe)        │         OpenProcess, NtReadVirtualMemory
└─────────────────────┘
         │
         │  Own window (transparent overlay)
         ▼
┌─────────────────────┐
│ RobloxPlayerBeta    │  DataModel → Workspace, Players, Camera, Parts…
│ (separate process)  │
└─────────────────────┘
```

**Internal** cheats inject a DLL and run code inside Roblox. Externals avoid injection but depend entirely on correct offsets and stable read paths.

---

## 2. Before you start

### Skills you need

| Area | Minimum | Why |
| --- | --- | --- |
| C++ | Comfortable with headers-only helpers, structs, Win32 | Most externals are C++ on Windows |
| Win32 | Processes, windows, messages, DPI | Attach + overlay |
| Linear algebra | Basic 3×3 / 4×4 matrix multiply | World-to-screen |
| Debugging | x64dbg or similar, logging | When offsets rot |
| Patience | Required | Roblox updates weekly |

### Environment

- **OS:** Windows 10/11 x64
- **IDE:** Visual Studio 2022+ with *Desktop development with C++* and CMake
- **Target game:** Roblox desktop (`RobloxPlayerBeta.exe`)
- **Network:** Needed once to fetch offset dumps; cache locally after that

### What this guide assumes you are *not* doing

- Shipping a frozen offset list inside your exe forever
- Ignoring Roblox deployment channels
- Reading memory every frame without caching or throttling discovery
- Drawing ESP before `Snap.ready` is true

---

## 3. How Roblox is laid out in memory

Roblox is a native C++ client. At runtime you care about a chain roughly like this:

```
Module base (RobloxPlayerBeta.exe)
    │
    ▼
FakeDataModel pointer          ← static RVA from offset dump
    │
    ▼
Real DataModel                 ← pointer at FakeDataModel + RealDataModel offset
    │
    ├── Workspace                ← services under DataModel
    ├── Players
    ├── Lighting
    └── …
            │
            ▼
        Player instances
            │
            ▼
        Character Model
            │
            ├── Humanoid       (health, walkspeed, …)
            └── Parts          (Head, HumanoidRootPart, limbs…)
```

Parallel to DataModel, many externals also read:

- **VisualEngine** — view matrix, screen dimensions
- **TaskScheduler** — jobs, FPS cap (advanced)
- **MouseService** — silent-aim style hooks (advanced, high risk)

Each arrow is an offset. If any offset is wrong for your build, everything downstream is garbage.

### Key structs (conceptual)

Your code typically defines mirrors like:

| Your struct | Purpose |
| --- | --- |
| `Actor` | One player/character worth of cached state for ESP/aim |
| `Snap` | One frame snapshot: view matrix, client rect, actor list |
| `Engine` | Process handle, base address, resolved pointers, offset table |
| `Off` | All numeric offsets loaded from JSON |

Keep game mirrors separate from UI state. The renderer should consume `Snap`, not walk Roblox memory while drawing.

---

## 4. The external pipeline

A maintainable external splits work into stages. FF0L follows this pattern:

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ offsets::Boot│───►│ world::Attach│───►│world::Resolve│───►│ world::Pulse │
│ fetch/cache  │    │ find process │    │ DataModel    │    │ build Snap   │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                                    │
                                                                    ▼
                                                          ┌──────────────┐
                                                          │ Draw / Aim   │
                                                          │ (per frame)  │
                                                          └──────────────┘
```

### Stage responsibilities

| Stage | Runs when | Success looks like |
| --- | --- | --- |
| **Boot offsets** | Once at startup (+ manual refresh) | `offsets::Ready()` true, version string known |
| **Attach** | Until process found | `OpenProcess` handle, module base, game HWND |
| **Resolve** | After attach, periodically | `DataModel`, `Workspace`, `Players` non-null |
| **Pulse** | Each frame when features need data | `Snap.ready == true`, `Snap.count > 0` |
| **Render** | Each overlay frame | ESP/FOV/menu drawn from `Snap` only |

**Rule:** If `Pulse` early-returns because no feature is enabled, attach may never run and the menu can look fine while the game path is dead. Gate features explicitly in UI copy and diagnostics.

---

## 5. Process attach and memory reads

### Finding Roblox

Scan for process names (newest clients use `RobloxPlayerBeta.exe`):

```cpp
static const wchar_t* Names[] = {
    L"RobloxPlayerBeta.exe",
    L"RobloxPlayer.exe",
    L"Windows10Universal.exe"
};
```

Use `CreateToolhelp32Snapshot` / `Process32FirstW` to get PID, then `OpenProcess`.

### Access levels

| Flags | Effect |
| --- | --- |
| `PROCESS_VM_READ \| PROCESS_QUERY_INFORMATION` | Read-only external (ESP, radar) |
| `+ PROCESS_VM_WRITE \| PROCESS_VM_OPERATION` | Movement, silent aim, noclip |

If `OpenProcess` fails with access denied, run as admin is **not** always the fix — protected processes and anti-cheat policies matter. Start read-only until attach is stable.

### Module base

Resolve `RobloxPlayerBeta.exe` base with `Module32FirstW` / `CreateToolhelp32Snapshot`. All static RVAs from offset dumps are **relative to this base**.

### Reading memory safely

Pattern every read through helpers:

1. Validate address (`>= 0x10000`, canonical user range)
2. Read via `ReadProcessMemory` or `NtReadVirtualMemory`
3. Track fail counts; detach if process exits

Never treat a non-null pointer as valid without range checks and occasional re-validation.

### Client version from the process path

Roblox installs per build under:

```
%LOCALAPPDATA%\Roblox\Versions\version-<hash>\RobloxPlayerBeta.exe
```

Parse `version-*` from `QueryFullProcessImageNameA` and compare to your offset dump version string. **Mismatch means do not trust any pointer chain** — see [Section 7](#7-the-live-channel-trap).

---

## 6. Offsets — the moving target

Offsets are numeric constants (RVAs and structure field displacements) produced by reverse engineering each Roblox build.

### Where dumps come from

FF0L uses [offsets.imtheo.lol](https://offsets.imtheo.lol):

| Endpoint | Returns |
| --- | --- |
| `/roblox/version` | Current LIVE version string, e.g. `version-f5a60436d48947d3` |
| `/offsets.json` | Full nested map: class → field → hex offset |

Example JSON shape (simplified):

```json
{
  "Roblox Version": "version-f5a60436d48947d3",
  "Offsets": {
    "FakeDataModel": { "Pointer": 147496136, "RealDataModel": 504 },
    "DataModel": { "Workspace": 344, "PlaceId": 400 },
    "Instance": { "ChildrenStart": 120, "NameContainer": 112 },
    "Player": { "LocalPlayer": 304, "ModelInstance": 664 }
  }
}
```

### Loading into your code

Map JSON into an `Off` struct at runtime:

```cpp
O.fakeDm = offsets::Get("FakeDataModel", "Pointer");
O.realDm = offsets::Get("FakeDataModel", "RealDataModel");
O.dmWorkspace = offsets::Get("DataModel", "Workspace");
O.playerLocal = offsets::Get("Player", "LocalPlayer");
// …
```

Never hardcode hundreds of offsets in source unless you enjoy rebuilding every patch.

### Caching strategy

1. On boot, HTTP GET live version
2. If version changed, download fresh JSON
3. Write `%AppData%\YourApp\offsets.json` + `offsets.version`
4. If offline, load cache and mark `stale`

Expose **Refresh** in settings and show the active hash in UI.

---

## 7. The LIVE channel trap

This is the most common “menu works, ESP doesn’t” bug for beginners.

### What Roblox channels are

Roblox ships multiple client builds (channels). Not every user runs the same `version-*` folder at the same time.

**Offset dump hosts typically dump only the LIVE (production) channel.**

If your installed client is:

```
version-241079e43bd84c46   ← your PC
```

But the dump says:

```
version-f5a60436d48947d3   ← LIVE dump
```

Then:

- Attach succeeds (you found the process)
- Offsets load (JSON parsed fine)
- **Resolve fails or returns nonsense** (FakeDataModel RVA is for a different binary)
- ESP/aim never get `Snap.ready`

Symptoms: watermark fine, menu fine, “resolving” or empty world forever.

### Fix — switch to LIVE with Fishstrap

Official walkthrough: [Switching to LIVE](https://offsets.imtheo.lol/docs/live-channel)

1. Download [Fishstrap](https://www.fishstrap.app/Fishstrap.exe)
2. Install and open **Fishstrap** from Windows search
3. **Configure Settings** → **Deployment**
4. Set **Channel** to `production`, press Enter
5. Set **Automatic channel change action** to **Never change**
6. **Save and Launch**

Always launch Roblox through Fishstrap after that. Verify your folder name matches the dump on [offsets.imtheo.lol/roblox/version](https://offsets.imtheo.lol/roblox/version).

### What your external should do

Compare client path version vs dump version every few seconds. On mismatch:

- Show a modal with the Fishstrap steps (do not silently fail)
- Disable ESP/aim until versions match
- Link to live-channel docs

FF0L implements this as `TickChannel()` + `DrawChannelNotice()`.

---

## 8. Resolving the game world

`Resolve()` turns offsets + base address into live pointers.

### Minimal resolve flow

```cpp
uintptr_t Fake = Read<uintptr_t>(base + off.fakeDm);
uintptr_t DataModel = Read<uintptr_t>(Fake + off.realDm);
if (!DataModel) return false;

uintptr_t Workspace = Read<uintptr_t>(DataModel + off.dmWorkspace);
if (!Workspace) Workspace = FindService(DataModel, "Workspace");

uintptr_t Players = FindService(DataModel, "Players");
uintptr_t LocalPlayer = Read<uintptr_t>(Players + off.playerLocal);
```

Cache `DataModel`, `Workspace`, `Players`. Re-resolve on a timer (400–800 ms) or when pointers go stale.

### VisualEngine

Optional but needed for accurate ESP projection:

```cpp
uintptr_t Visual = Read<uintptr_t>(base + off.visPtr);
Read view matrix from Visual + off.visView;
Read dimensions from Visual + off.visDim;
```

### Readiness flag

Set `Snap.ready = true` only when:

- Process attached
- DataModel + Players resolved
- At least discovery pass can run
- View matrix / client rect sane

UI and ESP must check `Snap.ready` before drawing.

---

## 9. Walking the instance tree

Roblox objects inherit from `Instance`. Children live in a contiguous array:

| Field | Role |
| --- | --- |
| `ChildrenStart` | Pointer to child pointer array |
| `ChildrenEnd` | End marker or count helper (dump-specific) |

`Kids()` typically tries two layouts (direct vs pointer-to-array) and caches `kidMode` once detected.

### Finding a service by name

```cpp
uintptr_t FindService(uintptr_t DataModel, const char* Want) {
    for each child :
        if ClassName(child) == Want) return child;
        if Name(child) == Want) return named;
    return 0;
}
```

Class names come from `ClassDescriptor` → `ClassName` string reads.

### Reading Roblox strings

Roblox uses small string optimization. Your reader should:

1. Read length at `Misc.StringLength` (often +16)
2. If length < 16, data inline; else pointer at object base
3. Reject non-printable garbage (bad offsets produce bad strings)

Cache `nameMode` once a valid name decode works.

---

## 10. Players, characters, and bones

### Discovery loop (throttled)

Do **not** full-scan every frame. FF0L uses ~700 ms discovery intervals:

1. Iterate `Players` children
2. Skip LocalPlayer for ESP targets
3. Read `ModelInstance` → character model
4. Find `Humanoid` + parts
5. Fill `Actor` struct, push to `Snap.list[]`

### R6 vs R15

Rigs use different part names. Maintain two name tables:

| Slot | R15 | R6 |
| --- | --- | --- |
| Head | Head | Head |
| Root | HumanoidRootPart | HumanoidRootPart |
| Torso | UpperTorso / LowerTorso | Torso |

Map aliases (`Left Leg` → multiple bone slots) for games that use simplified rigs.

### Health and filters

Read from Humanoid:

- `Health`, `MaxHealth`
- Skip if health ≤ 0
- Optional: team check, distance range, visibility rays

---

## 11. Camera, view matrix, and world-to-screen

### Client area vs overlay

Roblox window client rect gives screen origin of the game view:

```cpp
GetClientRect(gameHwnd);
ClientToScreen(gameHwnd, &origin);
Snap.clientX/Y/W/H = …
```

Your overlay is fullscreen. Convert game client coords → overlay coords with `ScreenToClient(overlayHwnd, …)`.

### Projection

Given view matrix `V` (4×4 or 3×4 layout — match your dump), world position `w`:

1. Transform to clip/NDC (implementation-specific)
2. Reject behind camera
3. Map to pixel coordinates inside `viewW × viewH`
4. Add `clientX/clientY` offset for full-screen overlay

Expose helpers:

- `world::ToView(worldPos, dot)` — engine space
- `world::ToScreen(worldPos, dot)` — screen pixels
- `EspDot(worldPos, overlayPoint)` — final draw space

### FOV circles

FOV radius in pixels is often derived from game client width/height and camera FOV, not overlay resolution. Recompute when Roblox window moves or resizes.

---

## 12. ESP — drawing what you read

ESP should be a **consumer** of `Snap`, not a scanner.

```cpp
void DrawEspWorld(float scale) {
    if (!Esp.on) return;
    const Snap& snap = world::View();
    if (!snap.ready || snap.count <= 0) return;

    for (int i = 0; i < snap.count; i++) {
        const Actor& a = snap.list[i];
        // project head/feet → box
        // draw box, name, health bar, skeleton lines
    }
}
```

### Feature gating

| Toggle | Must be true |
| --- | --- |
| ESP master | `Esp.on` |
| World data | `Snap.ready && Snap.count > 0` |
| Box visible | At least 2 projected points |
| Team filter | `!a.mate` when team check on |

### Color / visibility

Optional wall checks cast rays from camera to target head. Hysteresis (`visGood` / `visBad` counters) stops flicker.

---

## 13. Aim and input (high level)

Two common patterns:

| Type | Mechanism | Detectability |
| --- | --- | --- |
| **Mouse aimbot** | `SendInput` relative moves toward screen target | Moderate |
| **Silent aim** | Manipulate aim ray / mouse service in memory | High |

Design aim as:

1. Target selection (FOV, distance, bone, team, visibility)
2. Prediction (velocity × ping × distance factor)
3. Application (mouse move or memory write)

Keep selection in `TickAim()` separate from overlay drawing. Respect menu/listen modes so rebinding keys does not click targets.

**This guide does not provide hook signatures or bypass methods.** Treat write paths as high-risk and version-fragile.

---

## 14. Overlay UI architecture

FF0L uses a custom DirectX 11 overlay (`third_party/custom-framework`):

```
WinMain
  └─ ur::app::run(Config, Tick)
        ├─ CreateWindow (borderless, topmost, layered)
        ├─ DX11 swapchain (glass / DISCARD path for transparency)
        └─ each frame: Tick() → Engine render → Present
```

### Transparency paths (Windows)

Working combo for click-through overlays:

- `WS_EX_LAYERED` + `SetLayeredWindowAttributes`
- `DwmExtendFrameIntoClientArea({-1,-1,-1,-1})`
- DXGI swap effect `DISCARD`, BGRA format
- Clear alpha 0 in UI background; premultiply where required

Broken patterns (symptoms: black window or clicks eaten):

- Flip model + premultiplied alpha without full pipeline support
- `WS_EX_TRANSPARENT` without `WM_NCHITTEST` handling
- Using virtual desktop metrics instead of primary monitor for placement

### Menu vs world draw order

Typical frame when menu open:

1. Draw ESP / FOV on background route
2. Draw semi-transparent menu shell
3. Draw channel notice modal last (blocks click-through)

When menu closed:

- ESP + watermark only
- Click-through except draggable watermark chip

---

## 15. Click-through and hit-testing

Maintain a boolean `click_through`:

| Region | click_through |
| --- | --- |
| Over menu / modal | `false` |
| Over watermark (if draggable) | `false` |
| Empty overlay | `true` |

Toggle `WS_EX_TRANSPARENT` and call `SetWindowPos(..., SWP_FRAMECHANGED)` when changing modes.

Handle `WM_NCHITTEST` → return `HTTRANSPARENT` when click-through active so Windows forwards input to Roblox.

---

## 16. Threading and frame budget

| Work | Suggested thread | Interval |
| --- | --- | --- |
| Overlay render + input | Main thread | Every frame |
| Offset HTTP fetch | Background thread | On boot / refresh |
| Player discovery | Main thread (Pulse) | 500–700 ms |
| Resolve pointers | Main thread | 400–800 ms |
| Visibility rays | Main thread | When ESP vis check on |

Avoid blocking WinHTTP on the render thread during gameplay.

Cap reads per actor. ESP for 50 players with full skeleton every frame will spike CPU.

---

## 17. Project layout that scales

Example structure (based on FF0L):

```
src/
  Main.cpp          ← UI, menu, ESP draw, tick loop
  world.hpp         ← attach, resolve, pulse, projection
  offsets.hpp       ← HTTP fetch, JSON parse, cache
  silent.hpp        ← optional write features
  move.hpp          ← movement writes
  store.hpp         ← configs on disk
  sense.hpp         ← game-specific team/vis rules
assets/             ← fonts, icons (copied next to exe)
third_party/        ← UI framework, fonts
build.bat           ← one-click MSVC build
```

### Separation rules

- **offsets.hpp** — never includes rendering headers
- **world.hpp** — no UI types; pure data + Win32
- **Main.cpp** — orchestration only; keep individual draw tabs in functions

---

## 18. Debugging when nothing works

Use this decision tree:

```
Menu visible?
  NO  → overlay/window creation, DX init
  YES → Feature enabled (ESP/Aim toggle)?
          NO  → expected: Pulse may not attach
          YES → offsets::Ready()?
                  NO  → network, cache, JSON parse
                  YES → Client version == dump version?
                          NO  → LIVE channel / Fishstrap
                          YES → Snap.attached?
                                  NO  → Roblox not running / OpenProcess fail
                                  YES → Snap.ready?
                                          NO  → Resolve: DataModel/Players
                                          YES → Snap.count > 0?
                                                  NO  → discovery, wrong place
                                                  YES → projection / draw gate
```

### Log lines worth adding (dev builds)

- Offset version applied
- Client version from process path
- Resolve: Fake, DataModel, Players addresses (hex)
- Pulse: discovered count, ready flag
- First ESP projection failure reason

### Common mistakes

| Mistake | Symptom |
| --- | --- |
| Wrong channel | Attach OK, never ready |
| Stale offsets after patch | Random crashes, null Players |
| Drawing before `Snap.ready` | Flicker at 0,0 or empty |
| Using screen coords without client offset | ESP offset from game window |
| Full child scan every frame | Stutter, high CPU |
| Click-through always on | Cannot click menu |

---

## 19. Update survival checklist

When Roblox updates:

- [ ] Check [offsets.imtheo.lol/roblox/version](https://offsets.imtheo.lol/roblox/version)
- [ ] Confirm your client folder matches LIVE hash
- [ ] Refresh offsets in app or delete `%AppData%\YourApp\offsets.json`
- [ ] Re-test Resolve (DataModel, Players, LocalPlayer)
- [ ] Re-test ESP projection at multiple resolutions
- [ ] Re-test team check and rig types (R6/R15 map)
- [ ] Scan for renamed offsets in dump (ChildrenEnd, StringLength, etc.)

Ship version compare in production builds. Users should never guess why features silently died.

---

## 20. Tools worth learning

| Tool | Use |
| --- | --- |
| **Visual Studio + CMake** | Build/debug C++ overlay |
| **x64dbg** | Attach to Roblox, inspect pointers (ToS compliance: solo dev learning) |
| **IDA Pro / Ghidra** | Static analysis when dumps lag |
| **Process Hacker** | Verify handles, module base, command line |
| **Fishstrap** | Force LIVE channel |
| **offsets.imtheo.lol** | Versioned offset JSON |
| **RenderDoc** | DX11 overlay alpha debugging |

Learning order:

1. Read-only attach + print LocalPlayer name
2. Resolve Workspace + enumerate one child
3. Project HumanoidRootPart to screen, draw a dot
4. Build ESP box
5. Add menu + config system
6. Only then explore write features

---

## 21. Legal and safety notes

- Manipulating online games may violate Roblox [Terms of Use](https://en.help.roblox.com/hc/en-us/articles/203313410-Roblox-Community-Standards) and result in account action.
- Distributing cheats can carry legal exposure depending on jurisdiction.
- Reading or writing other processes can trigger anti-cheat or platform protections.
- This guide is for **educational understanding of Windows game client architecture**.

Build on private experiences. Do not harass developers or publish working bypasses targeting live player populations.

---

## 22. Further reading

| Resource | Topic |
| --- | --- |
| [offsets.imtheo.lol](https://offsets.imtheo.lol) | LIVE offset dumps |
| [Switching to LIVE](https://offsets.imtheo.lol/docs/live-channel) | Fishstrap / production channel |
| [FF0L source](https://github.com/ff0l/Roblox-external) | Reference external implementation |
| Microsoft Learn — `OpenProcess`, `ReadProcessMemory` | Win32 memory API |
| Microsoft Learn — `DwmExtendFrameIntoClientArea` | Glass overlay |

---

## Quick reference card

```
ATTACH     Find Roblox PID → OpenProcess → module base
OFFSETS    Fetch JSON for LIVE version → cache locally
VERSION    Parse version-* from process path → must match dump
RESOLVE    FakeDM → RealDM → Workspace + Players
PULSE      Discover actors → fill Snap → set ready
PROJECT    World → view matrix → screen → overlay coords
DRAW       if (ready && count) draw ESP; else skip
CHANNEL    mismatch → show Fishstrap modal, not silent fail
```

---

<p align="center"><sub>Guide maintained alongside FF0L development. Feedback and corrections welcome via issues on this repository.</sub></p>
