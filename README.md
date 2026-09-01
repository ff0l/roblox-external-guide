# Building Roblox Externals

How a Windows external actually talks to Roblox: attach, offsets, resolve, snapshot, then draw.

This is how [FF0L](https://github.com/ff0l/Roblox-external) is wired. Names below (`offsets::Boot`, `world::Attach`, `Snap`) are from that tree.

Roblox updates often. Offsets move. If your client is not on the LIVE channel, nothing after attach will make sense.

---

## Contents

**Part I — Setup**

1. [What an external is](#1-what-an-external-is)
2. [What you need](#2-what-you-need)

**Part II — The pipeline**

3. [The loop](#3-the-loop)
4. [Offsets](#4-offsets)
5. [Attach](#5-attach)
6. [The LIVE channel](#6-the-live-channel)
7. [Resolve](#7-resolve)
8. [The instance tree](#8-the-instance-tree)
9. [Pulse](#9-pulse)
10. [Players and bones](#10-players-and-bones)
11. [World to screen](#11-world-to-screen)
12. [ESP](#12-esp)
13. [Aim](#13-aim)

**Part III — The rest**

14. [The tick](#14-the-tick)
15. [Layout](#15-layout)
16. [Nothing works](#16-nothing-works)
17. [After an update](#17-after-an-update)

---

## Part I — Setup

## 1. What an external is

A separate process. It does not inject.

1. Find `RobloxPlayerBeta.exe`
2. `OpenProcess`
3. Use offsets for *this* Roblox build
4. Read DataModel / Players / parts / camera
5. Draw on your own overlay, or move the mouse

```
your exe  -- ReadProcessMemory / NtReadVirtualMemory -->  RobloxPlayerBeta
   |
   +-- own window (ESP, menu)
```

An internal loads a DLL into Roblox. An external lives outside and dies the moment offsets are wrong.

---

## 2. What you need

- Windows 10/11 x64
- C++ and Win32 (processes, windows, `ReadProcessMemory`)
- Enough matrix math to project a point
- Visual Studio 2022 + CMake, or whatever already builds the tree (`build.bat` in FF0L)

Target: desktop Roblox. Process names we look for:

```
RobloxPlayerBeta.exe
RobloxPlayer.exe
Windows10Universal.exe
```

---

## Part II — The pipeline

## 3. The loop

Do not walk Roblox memory from the draw function. Split it.

```
offsets::Boot     fetch / cache JSON
      |
world::Attach     PID, handle, module base
      |
world::Resolve    FakeDataModel -> DataModel -> Workspace, Players
      |
world::Pulse      fill Snap (actors, view, camera)
      |
draw / aim        read Snap only
```

In FF0L that is one `Tick()`:

```
offsets::Boot()
TickChannel()                          // client version vs dump version
world::Pulse(Want, NeedAim, Skel, ...)
TickAim(...)
```

`Pulse` only calls `Attach` / `Resolve` when something actually needs the world:

```
Want = Esp.on || Aim.on || Mute.on
```

If every feature is off, `Pulse` returns immediately. Menu still opens. Attach never runs. That is why a “working menu” and a dead ESP can happen at the same time — the toggles default off, and they live in collapsed folds.

`Snap` is the frame snapshot. `Actor` is one player on that snapshot. The overlay reads `world::View()`, it does not call `Kids()`.

---

## 4. Offsets

An offset is either:

- an RVA from `RobloxPlayerBeta.exe` (FakeDataModel pointer, VisualEngine pointer)
- a field displacement on a live object (Workspace on DataModel, Health on Humanoid)

They change every LIVE dump.

### Host

FF0L uses [offsets.imtheo.lol](https://offsets.imtheo.lol):

| Path | What you get |
| --- | --- |
| `/roblox/version` | LIVE version string, e.g. `version-f5a60436d48947d3` |
| `/offsets.json` | Nested map: class → field → number |

HTTPS, WinHTTP, TLS 1.2.

The JSON object has keys `Roblox Version`, `Dumped At`, `Total Offsets`, and `Offsets`. Under `Offsets` each class is an object of field names to integers. Do not paste last week’s numbers into source.

Load them at runtime:

```cpp
O.fakeDm      = offsets::Get("FakeDataModel", "Pointer");
O.realDm      = offsets::Get("FakeDataModel", "RealDataModel");
O.dmWorkspace = offsets::Get("DataModel", "Workspace");
O.playerLocal = offsets::Get("Player", "LocalPlayer");
```

`LoadOff()` maps the rest the same way (Instance children, Humanoid, Primitive, Camera, VisualEngine, …).

### Cache

`offsets::Sync`:

1. GET `/roblox/version`
2. Compare to `%AppData%\ff0l\offsets.version`
3. If it changed (or cache is missing), GET `/offsets.json`, parse, write `offsets.json` + `offsets.version`
4. If the network fails, load the last cache and mark it stale

`Boot()` runs `Sync` once on the first tick (this can stall that frame). A later Refresh uses `Request()`, which runs `Sync` on a detached thread.

If `offsets::Ready()` is false, `Resolve` has nothing to stand on.

---

## 5. Attach

`world::Attach()`:

1. If we already have a handle and the process is still alive, keep it (and `LoadOff` if offsets just became ready)
2. Otherwise wait 1500 ms between attempts
3. Snapshot processes, match one of the three exe names (`FindPid` keeps the last match)
4. `OpenProcess` — write+read first, then read-only, then `PROCESS_QUERY_LIMITED_INFORMATION` + read
5. Module base via `CreateToolhelp32Snapshot` / `Module32FirstW`
6. Find the game HWND
7. `LoadOff()`

Reads go through `Pull`. Address must pass `Heap` (`>= 0x10000` and below the user canonical top). Prefer `NtReadVirtualMemory` if `BindNt()` found it, else `ReadProcessMemory`. `Ptr()` is `Pull` + `Heap` on the value.

A non-null pointer is not a valid object. If Roblox exits, `GetExitCodeProcess` is not `STILL_ACTIVE` and we detach.

Write features need `PROCESS_VM_WRITE`. ESP does not.

---

## 6. The LIVE channel

This is the usual “menu works, ESP does not” case.

Roblox ships more than one client at a time. [offsets.imtheo.lol](https://offsets.imtheo.lol/docs/live-channel) only dumps **LIVE**.

Your install looks like:

```
%LOCALAPPDATA%\Roblox\Versions\version-<hash>\RobloxPlayerBeta.exe
```

`ReadClientVer` takes the process path (`QueryFullProcessImageNameA`) and cuts out `version-…`. `TickChannel` compares that to `offsets::CopyVersion()` every 2 seconds.

If they differ:

- Attach still works (you found a process)
- JSON still parses (you have *a* dump)
- `base + FakeDataModel` is the wrong RVA
- `Resolve` fails or returns junk
- `Snap.ready` stays false
- ESP / aim draw nothing

The app does **not** flip your toggles off. The world path just never becomes ready. FF0L opens a modal (`DrawChannelNotice`) with the Fishstrap steps. You can dismiss it. That does not fix the dump.

### Switch to LIVE

From [Switching to LIVE](https://offsets.imtheo.lol/docs/live-channel):

1. Download [Fishstrap](https://www.fishstrap.app/Fishstrap.exe) (from fishstrap.app — their GitHub was taken down)
2. Install it, open **Fishstrap** from Windows search
3. **Configure Settings**
4. **Deployment**
5. Channel: `production`, press Enter
6. Automatic channel change: **Never change**
7. **Save and Launch**

Launch Roblox through Fishstrap after that. The folder name should match [offsets.imtheo.lol/roblox/version](https://offsets.imtheo.lol/roblox/version).

---

## 7. Resolve

`Resolve()` turns base + offsets into live pointers.

```
Fake      = Ptr(base + off.fakeDm)
DataModel = Ptr(Fake + off.realDm)
Workspace = Ptr(DataModel + off.dmWorkspace)   // or FindService(..., "Workspace")
Players   = FindService(DataModel, "Players")
Local     = Ptr(Players + off.playerLocal)
Visual    = Ptr(base + off.visPtr)             // optional, for the view matrix
```

If Workspace is missing, FF0L also tries the parent of Workspace for Players.

`Resolve` returns true when `Players != 0`. It caches DataModel / Workspace / Players / Local / Visual. Re-run every 800 ms when Players, Workspace, and Local are all set, otherwise every 400 ms.

PlaceId (if the dump has it) is read here and handed to `sense::BindPlace` for game-specific team / vis rules.

If Fake or DataModel is 0, stop. Do not invent a fallback chain.

---

## 8. The instance tree

Children sit between `ChildrenStart` and `ChildrenEnd`.

`Kids()` tries two layouts, then remembers `kidMode`:

- **0** — start / end are on the instance
- **1** — `ChildrenStart` is a pointer to a small header, then start / end

Each layout tries stride 16, then stride 8. Span is capped. Empty or huge ranges are rejected.

`FindService` walks kids: class name first (`ClassDescriptor` → `ClassName`), then instance name.

### Strings

Roblox strings are SSO. `ReadRoblox`:

1. Length at `Misc.StringLength`, or 16 if that offset is missing
2. Length `< 16` → bytes sit on the object
3. Length `>= 16` → pointer at the object
4. Reject non-printable junk

Bad offsets produce garbage names. That is usually how you notice `NameContainer` or `StringLength` moved.

---

## 9. Pulse

`Pulse(Want, NeedAim, Skel, Range, NeedVis)` builds `Snap`.

| Arg | In FF0L |
| --- | --- |
| `Want` | ESP, aim, or silent/mute on |
| `NeedAim` | aim or mute on (extra bones) |
| `Skel` | `Esp.skeleton` |
| `Range` | ESP distance cap |
| `NeedVis` | ESP on, or aim/mute vis check |

If `Want` is false: `count = 0`, return. No attach.

Otherwise: `Attach` + `Resolve`. Missing workspace or players → `ready = false`, `count = 0`.

Then:

1. `Discover` at most every 700 ms (or immediately if nobody is tracked)
2. `ClientBox()` — Roblox client rect (`GetClientRect` + `ClientToScreen`)
3. View size from VisualEngine dimensions when they look sane (64–8000), else the client size
4. View matrix from VisualEngine
5. Camera position / right vector from `Workspace.CurrentCamera`
6. For each tracked actor: team, head/root, velocity, health, distance, ping, optional bones, optional wall check
7. Skip local (`self`), dead (`health <= 0.05`), out of range, or a character that failed `Heap`

`Snap.ready = true` after that path succeeds, **even if `count` is 0** (empty server, everyone filtered). ESP still requires `count > 0`.

Visibility uses cached wall parts plus hysteresis (`visGood` / `visBad`) so boxes do not flicker every frame.

Keep discovery off the per-frame path. Pulse already does that.

---

## 10. Players and bones

`Discover`:

1. `Kids(Players)`
2. Keep `Class == Player`
3. `ModelInstance` → character
4. Humanoid by class, then by name
5. DisplayName / instance name / humanoid name
6. `RigType` (non-zero treated as R15)
7. `BindBones`

Local is stored with `self = true` and dropped later in Pulse, not skipped during discovery.

### Rigs

Name tables:

| Slot | R15 | R6 |
| --- | --- | --- |
| Head | Head | Head |
| Root | HumanoidRootPart | HumanoidRootPart |
| Torso | UpperTorso / LowerTorso | Torso |
| Limbs | LeftUpperArm, … | Left Arm, Left Leg, … |

`BindBones` is not only those strings. It also:

- takes Humanoid.HumanoidRootPart / Model.PrimaryPart
- maps `Left Leg` / `Right Leg` onto several bone slots
- uses hitbox part names some games add
- if skeleton is on, assigns leftover low parts by side of the root

Games rename parts. If boxes exist and the skeleton looks drunk, the name table is the first place to look.

Hard cap: 64 actors (`ActorMax`).

---

## 11. World to screen

`ClientBox` writes `clientX/Y/W/H` from the Roblox HWND. If the window is gone, fall back to the primary screen metrics.

`Project` uses the 4×4 view matrix from VisualEngine:

```
X = w·row0 + M[3]
Y = w·row1 + M[7]
W = w·row3 + M[15]
if W < 0.05  → behind camera, skip
ndc → pixel in viewW × viewH
```

`ToScreen` then:

1. If VisualEngine size ≠ client size, scale
2. Add `clientX`, `clientY`

`EspDot` is `ToScreen` plus overlay-space conversion (`OverlayOf`). Draw in overlay pixels, not raw engine pixels.

FOV rings use the game client size and a screen-space radius around the aim midpoint, not the overlay’s full monitor size.

---

## 12. ESP

ESP only reads `Snap`.

```cpp
if (!Esp.on) return;
const Snap& snap = world::View();
if (!snap.ready || snap.count <= 0) return;
```

Then for each actor: range, team (`Esp.team && mate`), project head/feet/bones, box, name, health, skeleton.

No `ready` → no boxes. Wrong channel, failed resolve, or features off all look the same on screen.

---

## 13. Aim

Also a `Snap` consumer (`TickAim`).

1. Hold bind, menu not listening for a new key
2. Pick a target (FOV, team, vis, distance, bone, sticky)
3. Optional prediction from velocity / ping
4. Either relative mouse move, or a write into mouse/camera state (`silent.hpp`)

Writes are version-fragile. If you are learning the pipeline, get ESP on screen first.

This guide does not include write offsets, hooks, or bypasses.

---

## Part III — The rest

## 14. The tick

FF0L is a fullscreen DirectX 11 overlay (`ur::app::run`). One tick per frame.

Order that matters:

1. `offsets::Boot` — first call may hit the network
2. `TickChannel` — version compare, maybe open the Fishstrap modal
3. `world::Pulse` — only if a feature wants the world
4. `TickAim` / movement
5. Draw ESP from `Snap`, then the menu

Configs and the offset cache: `%AppData%\ff0l`.

---

## 15. Layout

```
src/
  Main.cpp       tick, menu, ESP draw, channel modal
  world.hpp      attach, resolve, pulse, project
  offsets.hpp    HTTP, parse, cache
  silent.hpp     optional writes
  move.hpp       movement writes
  sense.hpp      place-specific team / vis
  store.hpp      configs
assets/          fonts, icons (next to the exe)
third_party/     overlay
build.bat
```

`offsets.hpp` should not know about drawing. `world.hpp` should not know about menu widgets. `Main.cpp` consumes `Snap`.

---

## 16. Nothing works

```
Menu on screen?
  no  → overlay / D3D init
  yes → ESP or aim actually enabled?
          no  → Pulse never attaches. Open the fold, flip the switch.
          yes → offsets::Ready()?
                  no  → network, cache, JSON
                  yes → client version == dump version?
                          no  → Fishstrap, LIVE channel
                          yes → Snap.attached?
                                  no  → Roblox closed, or OpenProcess failed
                                  yes → Snap.ready?
                                          no  → Resolve (Fake / DataModel / Players)
                                          yes → Snap.count > 0?
                                                  no  → discovery, wrong place, everyone filtered
                                                  yes → ToScreen / draw
```

Worth printing once in a debug build: dump version, client version, Fake / DataModel / Players, `ready`, `count`.

| What you did | What you see |
| --- | --- |
| Wrong channel | Attach ok, never ready |
| Stale dump after a patch | Ready flapping, empty Players, junk names |
| Feature still off | Menu fine, `Pulse` no-ops |
| Drew before `ready` | Empty, or a box at 0,0 |
| Forgot client origin | ESP shifted vs the game window |
| `Kids` every frame | Stutter |

---

## 17. After an update

- [offsets.imtheo.lol/roblox/version](https://offsets.imtheo.lol/roblox/version)
- Your `Versions\version-*` folder matches that hash (Fishstrap if not)
- Refresh offsets, or delete `%AppData%\ff0l\offsets.json` and `offsets.version`
- Confirm Resolve: DataModel, Players, LocalPlayer
- Confirm one ESP box at your real resolution
- Check R6 / R15 and team on the place you actually play

Do not ship a frozen offset table in the exe.

---

Using this on live games can break Roblox’s terms and get accounts banned. Writing another process is also how you get your own tool killed by the next client change. Read the FF0L tree if you want the real control flow — this file is the map, not a cheat drop.

Reference: [ff0l/Roblox-external](https://github.com/ff0l/Roblox-external) · dumps: [offsets.imtheo.lol](https://offsets.imtheo.lol)

---

Text created with AI xd dont flame me for it 😭.
