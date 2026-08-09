# bt3recompiled

A static-recompilation project for **Dragon Ball Z: Budokai Tenkaichi 3** (Wii,
NTSC-US, title ID `RDSE70`), built on [DolRecomp](https://github.com/ExpansionPak/DolRecomp)
(a PowerPC/Gekko static recompiler) and [ModernGekko](https://github.com/ExpansionPak/ModernGekko)
(a Dolphin-derived runtime), following the layout of
[ModernGekko-Template](https://github.com/ExpansionPak/ModernGekko-Template).

The game's original PowerPC executable (`main.dol`) is translated ahead-of-time
into C, compiled to native x86-64, and run against a runtime that reimplements
the Wii's GPU/audio/IOS behavior (currently borrowed from Dolphin's own code,
with an in-progress effort to replace the GPU emulation layer with a native
D3D12 renderer written specifically for this game — see "Native rendering
pipeline" below).

## Current status (as of 2026-07-17)

- **CPU: fully static-recompiled, native, and stable.** The game boots and
  runs at ~60 FPS in a real window (headless mode also works). No interpreter
  or JIT executes at runtime — every PowerPC instruction was translated to
  real x86-64 code ahead of time by DolRecomp.
- **GPU/audio/IOS: currently emulated via the inherited Dolphin stack**
  (same as running the game in regular Dolphin, just with the CPU swapped
  out). This is the normal, working way to play the game today.
- **Native rendering pipeline: experimental, proof-of-concept stage.** A
  parallel, opt-in effort is underway to replace the emulated GX/Flipper GPU
  layer with a renderer that talks to D3D12 directly, using zero GPU hardware
  emulation. This does **not** yet replace the game's real rendering — it is
  a standalone research probe that has proven each piece of the pipeline
  works using real data captured from the game. See below for details and
  screenshots.

## Requirements

- Windows 10/11
- Visual Studio 2022 (MSVC + Ninja; the vendored Dolphin/ModernGekko C++ tree
  needs real MSVC — MinGW/GCC fails on many Windows-specific APIs it assumes)
- CMake ≥ 3.20
- Git (with submodule support)
- Your own legally-owned Wii ISO of Dragon Ball Z: Budokai Tenkaichi 3 (NTSC-US)

> DolRecomp itself builds fine with GCC/MinGW (pure C11), but the full
> ModernGekko/Dolphin C++ tree requires MSVC on Windows.

## Project layout

```
bt3recompiled/
├── lib/
│   ├── DolRecomp/            # submodule: PowerPC -> C static recompiler
│   └── ModernGekko/          # submodule: Dolphin-derived runtime + tools
│       └── vendor/dolphin/   # nested submodule: Dolphin fork (GXRuntime, VideoCommon, etc.)
├── iso/                      # put your own ISO here (gitignored)
├── extracted/<slug>/         # extracted disc contents (gitignored)
├── build/modules/            # compiled per-game native modules, cached by DOL hash (gitignored)
├── Makefile                  # tools / extract / recompile / run / clean targets
└── .gitmodules
```

## Building & running (the normal way)

1. **Rename your ISO** to avoid spaces/dashes (they break some of the tooling):
   e.g. `iso/BudokaiTenkaichi3.iso`.

2. **Build the tools** (DolRecomp + ModernGekko). On Windows this must be done
   through an MSVC developer environment, with MSYS2/MinGW stripped from
   `PATH` (its `pkg-config`/toolchain otherwise gets picked up by CMake and
   produces MinGW-incompatible dependency builds):

   ```bat
   call "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat"
   set PATH=%PATH:C:\msys64\ucrt64\bin;=%
   set PATH=%PATH:C:\msys64\usr\bin;=%
   set PATH=%PATH:C:\msys64\mingw64\bin;=%
   cmake -S lib\ModernGekko -B lib\ModernGekko\build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_SYSTEM_PROCESSOR=AMD64
   cmake --build lib\ModernGekko\build -j 4
   ```

   (`scratch_build_mg2.bat` at the project root wraps this exact sequence.)

3. **Extract the disc + recompile the DOL + run**, via `moderngekko-port`:

   ```bat
   lib\ModernGekko\build\moderngekko-port.exe extract "iso\BudokaiTenkaichi3.iso" --output "extracted\BudokaiTenkaichi3"
   lib\ModernGekko\build\moderngekko-port.exe build "extracted\BudokaiTenkaichi3" --output "build\modules"
   lib\ModernGekko\build\moderngekko-port.exe run "extracted\BudokaiTenkaichi3" --output "build\modules"
   ```

   Omit the last `-- --headless` argument to get a real window (default);
   add `-- --headless` to run without one. First `build` takes a while
   (full DOL decode + native compile); subsequent `run`s hit the module
   cache in `build/modules/RDSE70/<hash>/` and start immediately.

   There is currently no in-app controller-configuration UI — ModernGekko
   uses Dolphin's standard `GCPadNew.ini`/`WiimoteNew.ini` config files under
   the runtime's user directory if you need to map a real controller.

## How it works

- **DolRecomp** decodes the entire `main.dol` PowerPC binary ahead of time
  and emits one C file per code "chunk", plus a jump table, with no unknown
  opcodes for this title (871,733/871,736 recognized on the reference build).
  Each chunk is compiled into a shared library (`gRDSE70_recomp.dll`) tagged
  with an ABI version + CPU-state-size check so mismatched runtime/module
  pairs fail fast instead of silently corrupting memory.
- **ModernGekko** loads that module, sets `PowerPC::CPUCore::StaticRecomp` in
  a real Dolphin `Core::System`, and everything else (GX/Flipper GPU, DSP
  audio, IOS/HLE syscalls) runs through the same code paths as regular
  Dolphin — this project only replaced the CPU execution strategy, not the
  rest of the emulated machine, which is why the game already runs correctly
  today despite the native-renderer work below being unfinished.

## Native rendering pipeline (experimental)

The long-term goal of the `native-render-pipeline` branch is to remove the
GX/Flipper GPU **emulation** entirely — the same direction taken by
Zelda64Recomp-style projects — and replace it with a renderer that
reimplements this specific game's graphics as direct, native D3D12 calls.
This is *not* required for the game to run (see "Current status" above); it's
a from-scratch research effort layered alongside the working game.

### Why this is hard

BT3's graphics aren't simple. Real captured gameplay data (see
`gx_gameplay.log`, a 66 MB FIFO-command capture from a live combat) shows:

- ~1.18M draw calls per capture window, mostly `Quads`/`TriangleStrip`
- Almost all real geometry is submitted via **display lists** (20,773
  distinct addresses seen in one combat), not top-level draws
- Heavy multi-stage **TEV texture-combiner** usage (500+ distinct
  color/alpha-env states) — not simple single-texture shading
- Indirect texturing is rare (used for a specific effect, not pervasive)
- Real render-to-texture / EFB-copy usage (reflections or post-effects)

### What's been proven so far (each step verified against real captured
game data, and — critically — against actual rendered pixels via a
framebuffer readback, not just "no errors logged")

| Phase | What it proved |
|---|---|
| 0 | Captured real BT3 GX FIFO commands from live gameplay (`gx_gameplay.log`) |
| 1 | Dolphin's own shader-gen (`PixelShaderGen`/`VertexShaderGen`/`UberShader*`) produces correct GLSL from real captured TEV/BP state — reusable, no need to reimplement TEV→shader translation |
| 1b | That GLSL compiles end-to-end through the *real* Dolphin pipeline (glslang → SPIR-V → spirv-cross → HLSL) to real D3D bytecode |
| 1c | A real D3D12 device/swapchain/window renders *something* using that bytecode (first visible pixels, still a placeholder shader) |
| 2a | Fixed a VS/PS state-desync bug; the **real BT3-captured shader** (not a placeholder) successfully builds a D3D12 PSO and draws |
| 2b | Fixed the GX command decoder never having memory access, so display lists (where BT3 renders almost everything) were being silently skipped; real decoded vertex data (not synthetic) feeds the renderer |
| 2c | Fixed a `cpixelcenter` shader-uniform bug that was collapsing all geometry into a corner-pinned box regardless of source size |
| 3a/3b | Extended capture to multiple real draw calls merged into one scene fragment; began real (paletted) texture capture |
| 3c | Fixed the actual root cause of Phase 3's corner-pinning bug (same `cpixelcenter` issue, confirmed via pixel evidence: correctly-centered, properly-scaled geometry) |
| 4 | Resolved a real GX **C8 + TLUT (palette)** texture from live TMEM — genuinely decoded, not synthetic (happened to be a low-contrast menu/UI texture, not vivid combat art) |
| 5/5b | Raised merged draw-call capture from 16 to 300, added a capture-start delay to skip past the boot/menu flow — did not by itself fix the background/UI bias (see next row) |
| 6/6b/6c | Fixed it: filter out draws with uniform (RGB-only) vertex color, throttle the capture's RAM-refresh so it no longer visibly slows the game, broaden the filter to catch animated-alpha variants of the same flat overlay |
| 7/7b | Scan texture units 0-3 per draw instead of hardcoding unit 0, reject only fully-transparent decodes (a flat but *opaque* texture modulated by vertex color is normal, valid UI rendering, not something to discard) |
| 7c | Fixed a real bug: captured per-vertex color was parsed from the dump but then silently discarded, always fed as hardcoded white. Fixing it changed nothing visually — revealed the deeper Phase 8 issue below |
| 8 | **Architectural fix**: every draw was being rendered with ONE shared shader compiled from a single fixed reference state captured early on, regardless of what that draw's own real TEV/BP state actually was. Each of the 300 captured draws now compiles and renders with **its own real captured shader** — confirmed covering 300/300 draws with independent PSOs, no compile failures |
| 9 | **Root cause of the near-black output, fixed**: no real per-draw shader *constants* (TEV konst colors, alpha-test reference, blend mode, etc.) were ever computed from captured state — every prior phase fed the real shader code generic identity/`1.0f` cbuffer data instead. Reimplemented the relevant subset of Dolphin's real `PixelShaderManager` math directly from captured `bpmem`/`xfmem` (`BuildRealPixelConstants` in `dolphin_shader_compiler.cpp`) and wired it into the real per-draw PS cbuffer |
| 9b | Replaced the unreliable `MODERNGEKKO_GX_VERTEX_DUMP_SKIP_SECONDS` boot/menu-skip guess with F9-gated manual capture arming (`GxVertexDumpDevice::ToggleArmed()`, edge-triggered `GetAsyncKeyState(VK_F9)` polling in `OnRawFifoBytesForDump`) — the player presses F9 exactly when they reach real gameplay instead of guessing a fixed delay. First real interactive-combat capture with it: **11 distinct real shaders** across 180/300 draws, **4 distinct real textures** (256×256, 640×448, 4×4), and a real 3D bounding box (`x=[-359,32903] y=[-7874,448]`, vs. every prior capture's small near-planar UI-scale boxes) |

**Result:** the same capture that rendered near-black in Phase 8 now shows its real color — a bright teal/cyan (readback: avg=(9.9,56.0,69.8), max=(13,255,255), 23.67% non-background pixels), visibly matching the "flat teal-ish, 50% alpha" texture identified back in Phase 8 — see `phase9_frame0_readback.png`. The square and L-shaped bar are now genuinely rendering their real captured color, not just their real shape.

**Phase 9b result:** the first real interactive-combat capture (F9-armed mid-fight) renders a real, differently-shaped, multi-colored element (gray/cream/pink-accented diagonal shape, consistent with a combat effect like an energy trail or hit slash — see `phase9_combat_readback.png`), confirming Phase 9's real-constants fix also holds up against genuinely varied real gameplay data, not just the one reference capture used to diagnose it.

**Phase 9c**: a second, longer F9-armed combat capture (211/300 draws, 8 real shaders) rendered as one huge dominating shape again, hiding everything else — diagnosed as one draw's bounding box (real vertex-space span) being **72.5× larger** than the smallest draw's, since `LoadRealGeometry` was normalizing every merged draw into one *shared* NDC box (preserves real relative screen layout, but a huge draw's span swamps everything smaller down to invisibility). Changed to normalize each draw into its **own** NDC box instead (trades away relative real-world positioning for actually being able to see every draw's real shape — see `phase9c_perdraw_readback.png`, now showing several distinct overlapping shapes instead of one). A follow-up offline analysis of the raw capture then explained the *real* reason no character mesh has shown up yet: every one of the 211 captured draws (across both combat sessions so far) has only **3-5 vertices** — one triangle or quad — consistent with flat 2D UI (many have `span_z=0.0`, zero depth variation) or small VFX elements (auras/energy effects growing across a repeating draw cycle), not a real character mesh, which GX games typically submit as much larger multi-triangle batches per draw call. No character geometry has been *hidden*; none has been *captured* yet.

**Verification method:** since nobody driving this work can watch a live
window while it runs, every phase's proof is a framebuffer readback (backbuffer
→ CPU-readable buffer → `.ppm`/`.png` dump + pixel statistics), not just the
absence of D3D12 validation errors. This mattered: more than one bug in this
project produced *zero* D3D12 errors while rendering nothing/wrong, so
"no errors" was proven insufficient early on.

### Trying it yourself

```powershell
$env:MODERNGEKKO_REAL_GEOMETRY_DUMP = "D:\Proyectos\bt3recompiled\gx_vertex_dump.txt"
lib\ModernGekko\build\moderngekko_native_render_window.exe
```

This opens a ~640×480 window for ~12 seconds showing real decoded BT3
geometry, correctly positioned, drawn with the real compiled BT3 shader. As
of Phase 4 it should show a large, centered, mostly-dark rectangle (the
specific texture captured happened to be a near-black UI element — capturing
during actual combat, the way Phase 2b's data was gathered, would show
brighter character/effect textures instead).

To capture fresh gameplay data yourself (play a bit, then check the dump):

```powershell
$env:MODERNGEKKO_GX_VERTEX_DUMP = "D:\Proyectos\bt3recompiled\gx_vertex_dump.txt"
lib\ModernGekko\build\moderngekko-port.exe run "extracted\BudokaiTenkaichi3" --output "build\modules"
```

### What's next

- **Fixed in Phase 9**: real per-draw pixel-shader constants (see the phase
  table above). Note `VertexShaderConstants`/`FillIdentityAndOnes` for the
  VS stage was deliberately left untouched — the current bounding-box NDC
  positioning already works and depends on the VS cbuffer staying at its
  generic identity fill, so real VS constants are a separate, riskier piece
  of follow-up work, not a quick extension of the Phase 9 approach.
- **Done in Phase 9b**: a real interactive-combat capture through the
  per-draw-shader path, now with F9 arming instead of a guessed delay.
- Try to land on actual 3D character geometry instead of HUD/UI content
  — Phase 9c's per-draw analysis found every real draw captured so far
  (across two combat sessions) has only 3-5 vertices (flat UI or small VFX
  quads); a real character mesh is a much larger multi-triangle batch per
  draw call, so it hasn't been *hidden* by anything, it just hasn't been
  *captured* yet. Try a longer F9-armed session, or raise `m_max_draws`
  (currently 300, hardcoded in `MaybeEnableGxVertexDump`) so a session has
  more chances to include a real character draw before `Done()` cuts it
  off.
- Note for future capture sessions: `moderngekko-run.exe`'s default audio
  backend (WASAPI Exclusive Mode) takes exclusive control of the system
  audio device, silencing/blocking other apps (observed: YouTube, Discord)
  for as long as the game window is open. Pass `--audio "No Audio Output"`
  (as `scratch_launch_combat_capture.bat` now does) to avoid this.
- Once real character/effect art is actually captured: a real camera/
  projection setup, since the current bounding-box NDC normalization only
  works for near-planar/UI-style geometry
- Reimplementing more TEV combiner stages as real HLSL rather than relying
  on Dolphin's shadergen for every state
- Eventually retiring the Dolphin GX/VideoCommon dependency entirely once
  parity is reached — the actual "no more emulation" milestone

## Repository / branch notes

- `main` — the working, playable project (CPU-recompiled, Dolphin-backed
  rendering/audio/IOS)
- `native-render-pipeline` — the experimental work described above, kept
  separate since it doesn't yet replace anything in the working game
- `lib/DolRecomp` and `lib/ModernGekko` are real git submodules; `lib/ModernGekko`
  itself has a nested submodule at `vendor/dolphin` (a Dolphin fork).
  All three have their own `native-render-pipeline` branches with the commits
  referenced above.

## Credits

- [DolRecomp](https://github.com/ExpansionPak/DolRecomp) and
  [ModernGekko](https://github.com/ExpansionPak/ModernGekko) by ExpansionPak
- Built on [Dolphin](https://github.com/dolphin-emu/dolphin)'s codebase for
  the runtime/GPU/audio/IOS layer
- Project scaffold follows
  [ModernGekko-Template](https://github.com/ExpansionPak/ModernGekko-Template)
