VPORT technical overview
========================

How *Grand Theft Auto V* runs on iPhone, iPad and Apple silicon Macs. This page covers the engineering only. The figures
come from real runs, mostly on an iPhone 13 Pro (A15, 6 GB) and an M1 Mac, and describe the project as of November 2026.

[Back to the project page](README.md)

## Highlights

- **No emulator.** The game is a native ARM64 executable on every platform, with no x86 translation and no JIT.
- **A Windows runtime in one library.** The game still calls Windows. A compatibility runtime answers, covering roughly
  500 functions from 35 Windows DLLs.
- **Direct3D 11 on Metal, back to the A12.** The graphics layer picks one of three paths from the GPU family.
- **It fits in 3 GB.** On an iPhone XR, iOS allows about 1,850 MB and the game stays near 1,680 MB in play at the lowest
  settings.
- **Texture transcoding with no layout change.** Every BC block becomes an ETC2/EAC block of the same size. That alone
  took an A15 phone's memory footprint from 3.85 GB to 2.92 GB.
- **A 21.6 GB install on phones,** using ETC2/EAC textures, Zstandard resources and Opus audio.
- **Frames paced by the display.** At a 30 FPS limit on an M1 Mac, 300 of 300 frames reached the screen on time.
- **Touch controls driven by the game's own prompts,** so a new situation needs no hand-made layout.
- **Mods imported on the device,** with no PC: add-on packs, `.oiv` packages, maps, characters and languages.
- **.NET script mods on a platform with no JIT,** through a precompiled runtime.
- **Multiplayer with no central server.** A 12-character session code finds the host through the BitTorrent DHT, and UDP
  hole punching connects the players.
- **Crash reports that explain themselves.** Each one decodes the fault and records what the game was doing.

## The layers

The Windows version of the game expects x86-64, Win32, Direct3D 11, XInput and Bink video. VPORT keeps the game's
Direct3D 11 renderer, its PC settings and its game logic, and supplies everything below them.

| The game expects | VPORT provides |
|---|---|
| x86-64 code | ARM64 code, with no translation layer |
| Windows system DLLs | A compatibility runtime on Darwin (`libgtacompat.dylib`) |
| Direct3D 11 and DXGI | DXMT on Metal (`d3d11.dylib`, `dxgi.dylib`, `winemetal.dylib`) |
| BC-compressed textures | Sampled directly on Mac GPUs; ETC2/EAC on iPhone and iPad |
| Deflate-compressed resources | Deflate or Zstandard, told apart per resource |
| Compressed audio banks | Opus, at lower bitrates on phones |
| A Windows audio device | CoreAudio output for the game's software mixer |
| Bink video | HEVC and AAC, decoded by AVFoundation |
| A Win32 window and message loop | UIKit or AppKit with a `CAMetalLayer`; events become window messages |
| Keyboard, mouse and XInput | Touch layer, GameController, keyboard and mouse |
| Media Foundation (Self Radio) | AudioToolbox |
| DirectSound (voice chat) | AVAudioEngine |
| Online services | Player-hosted sessions, no central server |

## Compatibility runtime

`libgtacompat` plays the part of the Windows system DLLs. With the launcher, touch layer and network transport, which
live in the same library, it is about 48,000 lines of C++ and Objective-C++.

- **Threads and waits:** kernel objects and `WaitForMultipleObjects` (any or all) on per-thread Mach semaphores, events,
  mutexes, critical sections on `os_unfair_lock`, SRW locks, condition variables, suspended thread start, TLS and FLS.
- **Files:** Windows paths mapped into the app's folders.
- **Virtual memory:** `VirtualAlloc` on 16 KB pages with 64 KB reservations. `VirtualProtect` rounds inward to whole
  pages when it restricts access, so the 4 KB guard pages the game asks for can't take live memory with them.
- **Text:** a 16-bit `wchar_t` C runtime with MSVC's `printf` rules next to the platform's 32-bit one, and code pages
  mapped to UTF-8.
- **Windows and input:** window objects, the message queue, `WM_*` translation, raw input, monitors and display modes.
- **The rest:** an in-memory registry, COM types, shell folders, and a DXBC signature parser in place of `D3DReflect`.

**Process layout.** The first thread runs the UIKit or Cocoa event loop for the life of the process. The game runs on its
own threads and pumps its window messages from its render thread, and UI calls hop to the main queue. On iPhone and iPad
the game's thread only starts when Play is pressed in the launcher.

**Core scheduling.** iOS hands out the performance cores by QoS class. Only Windows' `TIME_CRITICAL` priority maps to
user-interactive, and the main thread, render thread and graphics threads ask for it. Before that, audio and particle
work outranked the frame's critical path on the A15's two performance cores.

**Details that had to match Windows exactly:**

- x86 locked instructions are full memory barriers. ARM64's atomic read-modify-writes let a later load pass their write,
  so a "queue a job, then check whether the worker is awake" handshake lost wake-ups, and the game froze with jobs queued
  and every worker asleep. Interlocked operations now have a full fence after them.
- Windows sets the first word of a `CRITICAL_SECTION` to -1. The game's lock wrapper skips locking while that word is 0,
  so with the word left at 0, no lock in the game locked anything.
- `ExitProcess` must not run static destructors, as on Windows. Running them crashed the game on exit.

## Graphics

Direct3D 11 reaches Metal through [DXMT](https://github.com/3Shain/dxmt), which translates the API calls and converts the
game's compiled DXBC shaders with an embedded LLVM. Converted shaders are cached on the device. VPORT carries a long list
of changes to DXMT, several of them starting from [an earlier iOS fork](https://github.com/willfaust/dxmt/tree/ios-port).
The iOS build uses UIKit instead of AppKit, drops macOS-only Metal APIs and keeps its shader cache in the app's caches
folder.

A hand-written Metal renderer was considered and rejected. Measured memory is dominated by the game's own data, so it
would save tens of megabytes at most, and the same shaders would still need converting.

### Three GPU paths

Apple's GPU families set the limits: tier 2 argument buffers start at the A13, mesh shaders at the A14, and BC textures
on iOS only at GPU family 9. The path is chosen at run time.

| GPU | Devices | What runs |
|---|---|---|
| A14, M1 and newer | iPhone 12 and later, iPads and Macs with those chips | Full path. Geometry shaders and tessellation run as Metal mesh shaders |
| A13 | iPhone 11 series, iPhone SE 2 | Direct3D 11 without tessellation or bokeh. No call names the object or mesh stages, because the A13's driver traps on them |
| A12, A12X, A12Z | iPhone XS and XR, iPad Pro 2018 and 2020 | Experimental. Direct3D 10 feature level, with argument tables written through `MTLArgumentEncoder` because these GPUs only have tier 1 |

The older paths leave the normal one alone: after the A12 work, shaders for the normal path compiled byte-identical
(362 of 362).

### Textures

Phone GPUs below family 9 can't sample BC. The first fallback decoded to RGBA8, at 2 to 8 times the BC size. VPORT instead
transcodes to formats whose blocks are exactly the size of the BC blocks, so the texture layout, mip chains and streaming
budgets stay the same and only the block contents change.

| BC format | Phone format |
|---|---|
| BC1 | ETC2 RGB8A1 |
| BC2, BC3, BC7 | EAC RGBA8 |
| BC4 | EAC R11 |
| BC5 | EAC RG11 |

- On the A15, the memory footprint went from 3.85 GB to 2.92 GB and Metal's share from 1.4 GB to 0.8 GB.
- Compared with Metal's own decode on an M1, quality is 34 to 40 dB for colour and 48 to 52 dB for alpha and one- or
  two-channel data. BC1 transparency is exact.
- The encoder takes 0.42 µs per BC1 block on one M1 core, and 0.48 to 1.18 µs for the other formats.
- Transcoding during streaming still cost 0.5 to 7 seconds of CPU per 10 seconds, which meant heat. Textures can also be
  converted ahead of time and marked so the graphics layer uploads them as they are, which takes that cost to zero.

### Frame pacing

The game's limiter sleeps on the CPU and then presents, so frames landed on whichever refresh came next. Now Present waits
until the previous frame is on screen (one frame deep at 40 FPS and below, two above), and the limiter finishes 1 ms early
so that the wait sets the phase. On an M1 Mac, 300 of 300 frames were on time at 30 FPS, and 601 of 601 at 60 FPS.

Two iOS surprises:

- Asking the display link for 24 Hz really ran the panel at 24 Hz. iOS then kept the performance cores nearly idle and the
  game fell to 18.6 FPS. The link now asks for the screen's top rate, with the limit as its floor.
- A minimum present duration of exactly 1/24 s sometimes took a sixth 120 Hz refresh, which is 20 FPS. It is now 2 ms
  shorter.

The Frame Limiter offers 24, 25, 30, 40, 45, 50, 60, 90, 120 and 144 FPS.

### Speed and heat

On phones the GPU produces the heat: at a 30 FPS cap the CPU drew 0.1 to 0.5 W while the phone stayed at thermal state
"serious".

- **Lossless render-target compression** on the A15 took an uncapped run at the same spot from about 43.4 to 51.8 FPS
  (+19%), and the G-buffer pass from 5.30 ms to 3.46 ms.
- **MetalFX spatial upscaling** of the presented frame, with its output capped at the screen's own pixels.
- **Temporal anti-aliasing** written for the layer as a compute shader: camera jitter, reprojected history through a 5-tap
  Catmull-Rom filter, and neighbourhood clipping in YCoCg.
- **A built-in per-pass profiler** lists one frame's passes with GPU time, draw count and megabytes loaded and stored.
- **CPU profiling** found a GUID parsed from text on every `QueryInterface`, 9% of the render thread, and a triangle check
  that took 26% of the encode thread. Both were fixed.

### Bugs only a port can have

- **Shaders wrong on iPhone only.** Xcode's C++ library implements `std::find` over 4-byte integers with `wmemchr`. On iOS
  the LLVM inside the shader converter bound that symbol to the compatibility runtime's 16-bit `wmemchr`, so searches
  found the wrong element and an optimisation pass dropped vector lanes. About 60% of converted shaders were subtly wrong
  and the frame came out dark. The converter now has a private 32-bit `wmemchr`. Loading the real iOS library on a Mac
  reproduced it.
- **Zero-area triangles.** The game joins quads into one triangle strip with repeated vertices. When the third vertex of
  such a triangle is behind the camera, the A15 rasterises a streak of pixels with garbage values. In the reflection cube
  those spread through the lighting until the whole frame was wrong. Non-indexed strips are now checked on the CPU, and
  those with such triangles are drawn as an indexed list without them.
- **Defined in Direct3D, undefined in Metal.** Render-target lanes a shader leaves unwritten and texture loads outside the
  texture now behave as Direct3D says. Non-finite values written to R11G11B10 or RGB9E5 targets become 0, so one bad pixel
  gives one black texel instead of a poisoned reflection map.
- **Blocky water.** These GPUs can't filter 32-bit float textures. The first fix made every sampler in a draw point-sample
  whenever one float texture was bound, and the ocean, which reads a float depth texture, turned its wakes into hard white
  blocks. Now only the sampler numbered like the float texture is affected.
- **Rivers eight times too strong.** The game reads river flow maps back from the GPU and samples them as DXT1. On phones
  the GPU holds those as ETC2, so the current pushed swimmers to one bank. That path now decodes ETC2, matching the GPU's
  own decode bit for bit.
- **A hang inside the driver on iOS 18.** Two threads inside the kernel's Metal resource creation at the same time could
  hang for good. As a workaround, creation below iOS 26 is now done one at a time, and readback buffers are pooled, which
  took the encode thread from 20 to 90 creations a second to almost none.

## Memory

iOS ends an app that passes its memory limit. The app carries the `increased-memory-limit` entitlement.

| Device | RAM | iOS allows | Measured in play |
|---|---|---|---|
| iPhone XR | 3 GB | about 1,850 MB | about 1,680 MB, peak 1,780 MB, lowest settings |
| iPad (10th generation) | 4 GB | about 2,870 MB | not measured |
| iPhone 13 Pro | 6 GB | about 4,095 MB | about 2,600 MB peak with High textures, shaders and reflections |

- **Decoded shaders.** DXMT kept the decoded form of all ~4,200 shaders for good, about 250 MB. They are now decoded only
  while a variant compiles, and the bytecode stays LZFSE-compressed (43 MB to 12 MB).
- **Low-memory profile.** On devices with little memory, lower budgets for the texture cache, streaming, level of detail,
  population and draw distance apply automatically. A shortened draw distance is hidden in haze.
- **Freed heap blocks** go back to the system through `madvise` and are reclaimed on reuse.
- **Graceful limits.** A map mod that overflows the object pool loses the extra objects instead of closing the game.

## Storage and data formats

The full install on phones is 21.6 GB, down from about 27 GB.

- **Zstandard resources.** Level 19 with a 2 MB window reached about 80% of the deflate size in samples, and took the main
  archives from 15.06 GB to 13.13 GB. It decodes as fast as zlib or faster. The loader checks each resource's first bytes
  for the zstd frame magic, so both formats coexist. Every recompressed frame is decoded and compared byte for byte.
- **Opus audio.** 32 kbps per channel for music, 24 kbps for speech, talk and DJ links. The 40 sound packs went from
  4.74 GB to 3.10 GB.
- **Video.** Bink movies become HEVC and AAC ahead of time. AVFoundation decodes them in hardware, and the planes go to the
  game's own YCrCb shader, so the colours don't change.

## Audio

- **Output:** a CoreAudio unit pulls float samples from the game's software mixer.
- **Interruptions:** iOS stops the output for a call, a timer or Siri, and the mixer used to wait on it forever, which
  froze the game. The mixer now runs on its own clock during an interruption, and the output is restarted afterwards or
  rebuilt after a media server reset.
- **Ring/Silent switch:** the audio session category keeps the game audible, and the volume buttons still work.
- **Self Radio:** AudioToolbox decodes MP3, AAC, WAV, FLAC, ALAC, AIFF and CAF.
- **Voice chat:** the game's DirectSound ring buffers are backed by AVAudioEngine. Range can be limited by distance, with
  push to talk or a toggle. The rings are overwritten as they go round, so nothing is recorded.

## Touch, controllers and keyboard

On iPhone and iPad, a view over the game acts as a virtual XInput pad and hides itself when a real controller connects.

- **Context.** The game tells the layer what the player is doing at every HUD update: on foot, driving, flying, cycling, in
  a menu, on the phone, at a weapon wheel or in a cutscene. Each context has its own layout.
- **Prompts.** The game also reports every pad button its on-screen prompts show. The layer highlights the buttons it has
  and adds a row for the rest, in the game's own words.
- **Minigames.** Some read a stick's position rather than its speed. When the prompts ask for a rotating stick, a real right
  stick appears.
- **Buttons appear only when they can do something,** such as Detonate with sticky bombs out or Enter with a car nearby.
- **Editing.** Controls can be moved and resized in the game. Each is stored as an offset from its default, so untouched
  ones follow later changes, and layouts can be shared as JSON.
- **Gyroscope steering** is optional. A 30-degree tilt is full lock, and a thumb on the stick takes over.
- **Controllers** connect through GameController, just after the game's first check for one, so the runtime sends the
  Windows device-change message the game listens for. Rumble reaches the pad, and phones can use haptics.
- **Keyboard and mouse** work on iPhone and iPad as well as on the Mac.

## Launcher

The launchers are native, UIKit on iPhone and iPad and AppKit on the Mac, and they open before the game starts.

- **Presets** follow the device's memory, and the GPU family decides which graphics path is offered.
- **Settings stay where the game keeps them.** The launcher reads the game's own files on open and writes only what changed
  on Play, keeping a backup of each file's first change.
- **Other pages** cover save import, the data scan, storage, thermal state and a summary of the last session from its log.
- **Languages** are JSON files dropped into a folder. Characters the game's fonts lack, such as Thai or Vietnamese, come
  from Core Text as outlines, shaped so that combining marks land in the right place.

## Mods

iOS only runs native code signed into the app, so a mod is either data the game reads or a script hosted by a runtime
already inside the app.

- **Add-on packs.** Each folder in the mods folder joins the game's DLC packs. A name starting with `_` or `.` switches it
  off, and deleting the folder uninstalls it.
- **Import on the device.** The launcher takes `.rpf`, `.zip`, `.rar`, `.7z` and `.oiv` files. Uncompressed or oversized
  textures are re-encoded to ETC2/EAC with the same encoders, and name clashes are resolved.
- **Maps and placements.** Map packs, plus the placement files community map editors produce.
- **ScriptHook-compatible host.** Its native parts ship in the app and stay inert until the pack is imported. Natives are
  resolved by their public hashes.
- **.NET mods.** ScriptHookVDotNet runs on Mono. iOS allows no JIT, so the class libraries are compiled ahead of time and
  the interpreter covers the rest. If the precompiled image fails to start, the host falls back to a smaller one.
  Microsoft's iOS runtime pack has only a stub for P/Invoke marshalling, so a mod passing a string to native code failed.
  VPORT links the real component.

## Multiplayer

Sessions are the game's own LAN sessions. A transport in the compatibility runtime makes distant players look like one
local network: each device gets a virtual 10.x.y.z address, and every packet travels over a single UDP socket.

- **Finding the host.** A host has a 12-character session code in Crockford base 32. Two names derived from it are used in
  BitTorrent's Mainline DHT, where the host announces itself and joiners look it up. On the same network, Bonjour finds it
  in about 0.1 s.
- **Connecting.** Each side learns its public address from STUN, asks the router for a port mapping, and tries every known
  address (LAN, IPv6, public, mapped) at the same time, so both NATs open. Addresses are tried every 300 ms for 15 s, then
  more slowly. The fastest path by round trip wins, is kept alive every 2 s and is dropped after 6.5 s of silence.
- **When no direct path exists,** the host relays for its players. An optional community server, a single Python script
  using only the standard library, adds relaying between strict networks, quick lookups and a public room list, with
  failover in 3.5 s.
- **Who can take part.** Every packet carries an HMAC-SHA256 tag keyed from the session code, so only code holders take
  part. DHT nodes and STUN servers see only a player's IP address.
- **The public DHT is hostile.** Many nodes near any name are crawlers that answer with made-up neighbours and swallow
  announcements, and some return poisoned addresses. Lookups now rely on nodes whose IDs pass the BEP 42 check, and the
  transport uses a random UDP port and ignores nodes that hand that port back. After that, joins by code took 3.5 to
  5.6 seconds in tests.

## Diagnostics and testing

- **Logs** have timestamps, a repeat limiter and a state line every 10 seconds. The app keeps the previous run's log, and
  each new log starts with how the last run ended.
- **Crash reports** decode the fault (read or write, null plus offset, the freed-memory fill pattern) and include the fatal
  message, memory and thermal state, annotated registers, the last second of game state, any exit call stack, load
  addresses for symbolication and the last 64 KB of the log.
- **Hang detection** dumps every thread's stack after 10 seconds and again at 60, including Metal resource creations in
  flight.
- **Testing without a device.** The launcher and every touch layout render off-screen to PNG, and the network transport
  runs without the game against emulated NATs.
- **A/B runs on real hardware** use the same save, the same spot and a switch for each change.
- **Support** runs on a knowledge base of more than 100 entries that map log signatures to causes and fixes.

## Devices

| Hardware or system | Status |
|---|---|
| A14 and newer, 6 GB or more | Confirmed by players, iPhone 12 Pro Max through iPhone 17 Pro Max and iPad Pro M4 |
| A14 and A15, 4 GB | Lowest preset: iPhone 12, iPhone 13, iPad (10th generation) |
| A13 | Confirmed on an iPhone 11 |
| A12, A12X, A12Z | Experimental. Confirmed on an iPhone XR and an A12Z iPad Pro |
| A11 and older | Not supported |
| iOS | 18 and later confirmed, 17 untested. 16 doesn't work: its GPU compiler refuses every pipeline |
| Mac | Apple silicon, M1 or later, macOS 14 or later |

## Built with

- [DXMT](https://github.com/3Shain/dxmt) and [its iOS fork](https://github.com/willfaust/dxmt/tree/ios-port)
- [LLVM](https://github.com/llvm/llvm-project), inside the shader converter
- [Opus](https://github.com/xiph/opus) and [Zstandard](https://github.com/facebook/zstd)
- [.NET runtime and Mono](https://github.com/dotnet/runtime), and [ScriptHookVDotNet](https://github.com/scripthookvdotnet/scripthookvdotnet)
- Apple's Metal, MetalFX, UIKit, AppKit, GameController, AVFoundation and CoreAudio

---

Grand Theft Auto is a trademark of Take-Two Interactive Software. This project is not affiliated with or endorsed by
Rockstar Games or Take-Two.
