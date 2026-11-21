VPORT technical overview
========================

How *Grand Theft Auto V* runs on iPhone, iPad and Apple silicon Macs. This page covers the engineering only. The figures
come from real runs, mostly on an iPhone 13 Pro (A15, 6 GB), an iPhone 6s (A9, 2 GB) and an M1 Mac, and describe the
project as of 21 November 2026.

[Back to the project page](README.md)

## Highlights

- **No emulator.** The game is a native ARM64 executable on every platform, with no x86 translation and no JIT.
- **A Windows runtime in one library.** The game still calls Windows. A compatibility runtime answers, covering roughly
  500 functions from 35 Windows DLLs.
- **A native Metal renderer.** The game's own renderer draws straight through Metal, with no Direct3D layer in between.
  DXMT, which translates Direct3D 11 to Metal, stays in the app as the alternative.
- **Shaders compiled ahead of time.** About 9,100 Metal libraries ship inside the app, so nothing is converted while
  playing.
- **It fits in 2 GB.** On an iPhone 6s, iOS allows 1,449 MB and the game peaked at 1,323 MB in play at the lowest
  settings.
- **Back to the A9 and iOS 15.** GPUs older than the A13 run the game at the Direct3D 10 feature level.
- **Texture transcoding with no layout change.** Every BC block becomes an ETC2/EAC block of the same size. That alone
  took an A15 phone's memory footprint from 3.85 GB to 2.92 GB.
- **A 21.6 GB install on phones,** using ETC2/EAC textures, Zstandard resources and Opus audio.
- **Frames paced by the display.** At a 30 FPS limit on an M1 Mac, 300 of 300 frames reached the screen on time.
- **Touch controls driven by the game's own prompts,** so a new situation needs no hand-made layout.
- **Mods imported on the device,** with no PC: add-on packs, `.oiv` packages, maps, characters, languages and script mods,
  including Windows `.asi` plugins, which run in an interpreter.
- **.NET script mods on a platform with no JIT,** through a precompiled runtime.
- **GTA Online's own scripts, run locally.** Local stand-ins answer for Rockstar's online services.
- **Multiplayer with no central server.** A 12-character session code finds the host through the BitTorrent DHT, and UDP
  hole punching connects the players.
- **Crash reports that explain themselves.** Each one decodes the fault and records what the game was doing.

## The layers

The Windows version of the game expects x86-64, Win32, Direct3D 11, XInput and Bink video. VPORT keeps the game's renderer
(its Direct3D 11 feature set and PC data layouts), its PC settings and its game logic, and supplies everything below them.

| The game expects | VPORT provides |
|---|---|
| x86-64 code | ARM64 code, with no translation layer |
| Windows system DLLs | A compatibility runtime on Darwin (`libgtacompat.dylib`) |
| Direct3D 11 and DXGI | A native Metal renderer inside the game's renderer; DXMT on Metal as the alternative |
| Shaders compiled for Direct3D | Metal libraries compiled ahead of time, in a pack inside the app |
| BC-compressed textures | Sampled directly on Mac GPUs; ETC2/EAC on iPhone and iPad |
| Deflate-compressed resources | Deflate or Zstandard, told apart per resource |
| Compressed audio banks | Opus, at lower bitrates on phones |
| A Windows audio device | CoreAudio output for the game's software mixer |
| Bink video | HEVC and AAC, decoded by AVFoundation |
| A Win32 window and message loop | UIKit or AppKit with a `CAMetalLayer`; events become window messages |
| Keyboard, mouse and XInput | Touch layer, GameController, keyboard and mouse |
| Media Foundation (Self Radio) | AudioToolbox |
| DirectSound (voice chat) | AVAudioEngine |
| Online services | Local stand-ins, and player-hosted sessions with no central server |

## Compatibility runtime

`libgtacompat` plays the part of the Windows system DLLs. With the launcher, touch layer and network transport, which
live in the same library, it is about 60,000 lines of C++ and Objective-C++.

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

**Two-core devices.** The game's loading waits spin, as they do on Windows, and its loading screen drew at 63 FPS. On the
iPhone 6s both cores stayed busy, the streaming threads got almost no time, and the first load took about 25 minutes,
part of it one-time cache work. On devices with two cores the waits now sleep for 1 ms and loading screens run at
15 FPS at most: Play to in-game takes about 50 seconds.

**Details that had to match Windows exactly:**

- x86 locked instructions are full memory barriers. ARM64's atomic read-modify-writes let a later load pass their write,
  so a "queue a job, then check whether the worker is awake" handshake lost wake-ups, and the game froze with jobs queued
  and every worker asleep. Interlocked operations now have a full fence after them.
- Windows sets the first word of a `CRITICAL_SECTION` to -1. The game's lock wrapper skips locking while that word is 0,
  so with the word left at 0, no lock in the game locked anything.
- `ExitProcess` must not run static destructors, as on Windows. Running them crashed the game on exit.
- `VirtualQuery` must describe a gap in the address space as one free region and fail past its end. On machines under
  2.5 GB, a PC code path adds up the free address space with it, and on the first 2 GB device that walk never ended: the
  game hung at its first streaming allocation.

## Graphics

The game's renderer stays: RAGE's PC renderer, with its Direct3D 11 feature set and the PC data layouts. Below it, VPORT
has two ways to reach Metal, picked in the launcher's Renderer setting.

- **The native Metal renderer,** the default since 13 November 2026, is about 23,000 lines inside the game. The
  renderer's GPU objects are Metal-backed objects of the same pointer size, so the PC resource files load unchanged, and
  the render thread's draws reach Metal with no Direct3D call in between.
- **[DXMT](https://github.com/3Shain/dxmt)** translates the Direct3D 11 API to Metal and converts the game's compiled DXBC
  shaders with an embedded LLVM. It drew the game until November. VPORT carries a long list of changes to it, several of
  them starting from [an earlier iOS fork](https://github.com/willfaust/dxmt/tree/ios-port).

Why a renderer of its own: the game knows when its targets are locked, which resources an effect binds and when a buffer
is rewritten, which a translation layer has to work out again on every draw. A hot iPhone's frame is bound by the render
thread on the efficiency cores, so that work matters. A renderer of its own can also encode on a thread of its own, load
shaders compiled ahead of time, and run on GPUs and iOS versions that DXMT can't.

### Shaders

- **Compiled ahead of time.** Every vertex, pixel and compute program is converted offline to SPIR-V, then to Metal
  Shading Language with SPIRV-Cross, and Apple's offline compiler turns the result into Metal libraries. The app carries
  them in a 60 MB pack, compressed with Zstandard against a dictionary trained on the libraries.
- **No conversion at run time.** The pack also holds each program's bindings. A shader loads in 0.01 to 0.04 ms, where
  compiling one at first use took 18 to 30 ms on average, and nothing depends on iOS keeping a shader cache, which it
  empties when storage runs low.
- **Argument tables.** A stage's textures, samplers and constant buffers go into Metal argument tables instead of a
  separate bind each.
- **DXMT's converter** is kept for the depth-of-field bokeh of the highest Post FX settings, on GPUs that can draw it.

### Encoding

- **An encode thread.** The render thread records its drawing calls into a 32 MB ring, and a second thread turns them
  into Metal commands. On a hot iPhone 13 Pro the dense city's render thread went from 93-94% of a core to 57-65%, and
  FPS rose 1.2% to 5.4%.
- **Metal 4 (BETA)** runs the same renderer on Apple's newer API, on iOS or macOS 26 with an A14, M1 or later.

### GPU paths

Apple's GPU families set the limits: tier 2 argument buffers start at the A13, mesh shaders at the A14, and BC textures
on iOS only at GPU family 9. The path is chosen at run time.

| GPU | Devices | What runs |
|---|---|---|
| A14, M1 and newer | iPhone 12 and later, iPads and Macs with those chips | Everything. With DXMT, geometry shaders and tessellation run as Metal mesh shaders |
| A13 | iPhone 11 series, iPhone SE 2 | The native renderer in full. DXMT without tessellation or bokeh |
| A9 to A12 | iPhone 6s through XS and XR, iPads from the A9 to the A12Z | Direct3D 10 feature level. These GPUs only have tier 1 argument buffers, so the tables are written through `MTLArgumentEncoder`. DXMT only on the A12, experimental |

iOS 15 has no GPU addresses for resources, no MetalFX and no Metal 3 shading language. There the native renderer writes
MSL 2.4, binds resources directly and takes every shader from the pack. DXMT is offered from iOS 17.

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
- Transcoding during streaming still cost 0.5 to 7 seconds of CPU per 10 seconds, which meant heat. The phone data set is
  now converted ahead of time and marked so the renderer uploads it as it is, which takes that cost to zero.

### Frame pacing

The game's limiter sleeps on the CPU and then presents, so frames landed on whichever refresh came next. Now Present waits
until the previous frame is on screen (one frame deep at 40 FPS and below, two above), and the limiter finishes 1 ms early
so that the wait sets the phase. On an M1 Mac, 300 of 300 frames were on time at 30 FPS, and 601 of 601 at 60 FPS. Both
renderers pace this way.

Two iOS surprises:

- Asking the display link for 24 Hz really ran the panel at 24 Hz. iOS then kept the performance cores nearly idle and the
  game fell to 18.6 FPS. The link now asks for the screen's top rate, with the limit as its floor.
- A minimum present duration of exactly 1/24 s sometimes took a sixth 120 Hz refresh, which is 20 FPS. It is now 2 ms
  shorter.

**Lower FPS When Hot.** When a device gets hot and can't keep up with the limiter, the game steps down to 40 FPS (30 where
40 doesn't divide the screen's rate, or in Low Power Mode) and back up once it cools. On a hot iPhone 13 Pro that gave a
steady 39.8 to 40.0 FPS, where a 60 FPS limit swung between 45 and 57.

The Frame Limiter offers 24, 25, 30, 40, 45, 50, 60, 90, 120 and 144 FPS.

### Speed and heat

On phones the GPU produces the heat: at a 30 FPS cap the CPU drew 0.1 to 0.5 W while the phone stayed at thermal state
"serious".

- **Upscaling.** On iPhones a single sharpening pass enlarges the frame by default. On a hot iPhone 13 Pro it gave 11.6%
  more FPS than MetalFX, whose spatial scaler costs 3 to 4 ms of GPU a frame at the screen's 3 megapixels. MetalFX stays
  the default on iPads and Macs, with its output capped at the screen's own pixels.
- **Lossless render-target compression** on the A15 took an uncapped run at the same spot from about 43.4 to 51.8 FPS
  (+19%), and the G-buffer pass from 5.30 ms to 3.46 ms.
- **Less work per frame.** Reflections render two of the cube's six faces a frame (+3.0% FPS on the 13 Pro). At night, the
  fog volumes' converted triangle strips are cached and the render thread gets transient memory of its own (+9.0% FPS on
  the M1, +1.1% on the GPU-bound 13 Pro). Depth the renderer only reads is sampled in place instead of copied (+3% on the
  M1).
- **Temporal anti-aliasing** written as a compute shader: camera jitter, reprojected history through a 5-tap Catmull-Rom
  filter, and neighbourhood clipping in YCoCg.
- **Built-in profilers** list one frame's passes with GPU time, draw count and megabytes loaded and stored, and break the
  render thread's time down per draw.
- **CPU profiling** found a GUID parsed from text on every `QueryInterface`, 9% of the render thread, and a triangle check
  that took 26% of the encode thread. Both were fixed.

### Bugs only a port can have

- **Shaders wrong on iPhone only.** Xcode's C++ library implements `std::find` over 4-byte integers with `wmemchr`. On iOS
  the LLVM inside DXMT's shader converter bound that symbol to the compatibility runtime's 16-bit `wmemchr`, so searches
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
- **A freed vertex buffer.** In a path of the PC game, a draw could be set up with the previous draw's vertex buffer, which
  might already be freed. Fixed on every Apple build.
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
| iPhone 6s | 2 GB | about 1,450 MB | peak 1,323 MB, lowest settings |
| iPhone XR | 3 GB | about 1,850 MB | about 1,680 MB, peak 1,780 MB, lowest settings |
| iPad (10th generation) | 4 GB | about 2,870 MB | not measured |
| iPhone 13 Pro | 6 GB | about 4,095 MB | about 2,600 MB peak with High textures, shaders and reflections |

- **Decoded shaders.** DXMT kept the decoded form of all ~4,200 shaders for good, about 250 MB. They are now decoded only
  while a variant compiles, and the bytecode stays LZFSE-compressed (43 MB to 12 MB). With the shader pack, the native
  renderer keeps no bytecode for the shaders it serves.
- **Low-memory profile.** On devices with little memory, lower budgets for the texture cache, streaming, level of detail,
  population and draw distance apply automatically. A shortened draw distance is hidden in haze.
- **The 2 GB profile** goes further on devices with 2.5 GB of RAM or less. The resident audio banks (3,449 sounds of
  16-bit PCM, 108 MB) are re-encoded at load as IMA ADPCM, which saves 75 MB. Textures load two mip levels lower, object
  pools and some render targets are smaller, and 16 damaged vehicles are kept at a time instead of the PC's 40. With the
  whole map drawn, the benchmark's peak on the Mac fell from 1,607 to 1,341 MB.
- **Buffers on first write.** The game creates about 3,500 dynamic buffers and writes 100 to 160 of them in a session. On
  2 GB devices each gets its memory at its first write, which took upload memory from about 70 MB to 14-29 MB.
- **One tested set for 2 GB devices:** 800 × 450 with the 3D scene drawn at half that, 24 FPS and the lowest detail, as
  played on the iPhone 6s. Every preset applies it, and players can raise any setting themselves.
- **Freed heap blocks** go back to the system through `madvise` and are reclaimed on reuse.
- **Graceful limits.** A map mod that overflows the object pool loses the extra objects instead of closing the game.

## Storage and data formats

The full install on phones is 21.6 GB, down from about 27 GB. The app itself carries the 60 MB shader pack.

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
- **Buttons appear only when they can do something,** such as Detonate with sticky bombs out, or Enter beside a car,
  including one a mission seats the player in.
- **Editing.** Controls can be moved and resized in the game. Each is stored as an offset from its default, so untouched
  ones follow later changes, and layouts can be shared as JSON.
- **Gyroscope steering** is optional. A 30-degree tilt is full lock, and a thumb on the stick takes over.
- **Controllers** connect through GameController, just after the game's first check for one, so the runtime sends the
  Windows device-change message the game listens for. Rumble reaches the pad, and phones can use haptics.
- **Keyboard and mouse** work on iPhone and iPad as well as on the Mac.

## Launcher

The launchers are native, UIKit on iPhone and iPad and AppKit on the Mac, and they open before the game starts.

- **Presets** follow the device's GPU class and the memory iOS gives the game. On 2 GB devices every preset is the one
  tested set.
- **Renderer:** Metal 3, Metal 4 (BETA) or DXMT, each offered only where it runs.
- **Settings stay where the game keeps them.** The launcher reads the game's own files on open and writes only what changed
  on Play, keeping a backup of each file's first change. Graphics settings export and import as JSON files.
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
- **Windows `.asi` plugins** run in an x86-64 interpreter inside the app, with no JIT. A plugin's features that patch the
  PC game's memory stay off. ELS 1.05 runs from its original files, with AdvancedHookV built into the host.
- **.NET mods.** ScriptHookVDotNet runs on Mono. iOS allows no JIT, so the class libraries are compiled ahead of time and
  the interpreter covers the rest. If the precompiled image fails to start, the host falls back to a smaller one.
  Microsoft's iOS runtime pack has only a stub for P/Invoke marshalling, so a mod passing a string to native code failed.
  VPORT links the real component.

## Multiplayer

### GTA Online's own scripts

The GTA ONLINE page starts the game's own GTA Online scripts: the character creator, the tutorial, free roam, the phone,
shops, properties and jobs.

- **Local stand-ins.** Every `rockstargames.com` address is blocked at name lookup. Local stand-ins answer for sign-in,
  game settings (tunables), transactions, presence and profile stats, and characters are saved on the device.
- **Jobs.** 424 of Rockstar's jobs are listed offline from the game's own data, and a race starts and runs.
- **Players** join a session with a code, over the transport below.

### The transport

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

- **Logs** have timestamps, a repeat limiter and a state line every 10 seconds. The app keeps the previous three runs'
  logs, and each new log starts with how the last run ended. A Detailed Logging switch adds frame statistics, stack dumps of
  stalls and the game's own messages.
- **Crash reports** decode the fault (read or write, null plus offset, the freed-memory fill pattern) and include the fatal
  message, memory and thermal state, annotated registers, the last second of game state, any exit call stack, load
  addresses for symbolication and the last 64 KB of the log.
- **Hang detection** dumps every thread's stack after 10 seconds and again at 60, including Metal resource creations in
  flight.
- **Testing without a device.** The launcher and every touch layout render off-screen to PNG for any device model and
  memory size, and the network transport runs without the game against emulated NATs.
- **A/B runs on real hardware** use the same save, the same spot and a switch for each change, or switch between the two
  arms inside one run, so a hot phone's falling clock affects both alike.
- **Support** runs on a knowledge base of more than 100 entries that map log signatures to causes and fixes.

## Devices

| Hardware or system | Status |
|---|---|
| A14 and newer, 6 GB or more | Confirmed by players, iPhone 12 Pro Max through iPhone 17 Pro Max and iPad Pro M4 |
| A14 and A15, 4 GB | Lowest preset: iPhone 12, iPhone 13, iPad (10th generation) |
| A13 | Confirmed on an iPhone 11 |
| A12, A12X, A12Z | Confirmed on an iPhone XR and an A12Z iPad Pro |
| A10, A11 | Confirmed on an A10 iPad and an iPhone 8 Plus |
| A9, 2 GB | Played on an iPhone 6s with iOS 15.8.3: the prologue and the first mission, at about 23 FPS |
| A8 and older | Not supported |
| iOS | 15 and later. iOS 15 runs the native renderer only; DXMT is offered from iOS 17 |
| Mac | Apple silicon, M1 or later, macOS 14 or later |

## Built with

- [DXMT](https://github.com/3Shain/dxmt) and [its iOS fork](https://github.com/willfaust/dxmt/tree/ios-port)
- [DXVK](https://github.com/doitsujin/dxvk)'s DXBC compiler and [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross),
  for the shader pack
- [LLVM](https://github.com/llvm/llvm-project), inside DXMT's shader converter
- [Opus](https://github.com/xiph/opus) and [Zstandard](https://github.com/facebook/zstd)
- [.NET runtime and Mono](https://github.com/dotnet/runtime), and [ScriptHookVDotNet](https://github.com/scripthookvdotnet/scripthookvdotnet)
- [Blink](https://github.com/jart/blink), the x86-64 interpreter for `.asi` plugins
- Apple's Metal, MetalFX, UIKit, AppKit, GameController, AVFoundation and CoreAudio

---

Grand Theft Auto is a trademark of Take-Two Interactive Software. This project is not affiliated with or endorsed by
Rockstar Games or Take-Two.
