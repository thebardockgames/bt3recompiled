# Handoff notes for the next AI session

This file is for an AI assistant picking up this project cold. It's a
technical brain-dump of state, gotchas, and next steps — not user-facing
documentation (see `README.md` for that). Read this fully before touching
anything; several of the gotchas below cost real time to rediscover once.

Last updated: 2026-07-17, end of a single long session that took this
project from "nothing" to a verified native-D3D12 rendering proof-of-concept.
Branch: `native-render-pipeline` (root + `lib/ModernGekko` + its nested
`lib/ModernGekko/vendor/dolphin`, all three on that branch, all three have
their own independent commit history — see "Repo topology" below).

## Repo topology — READ THIS FIRST

This is **not** a single repo. It's:

- `D:\Proyectos\bt3recompiled` — root project, its own git repo, `main` +
  `native-render-pipeline` branches
- `lib/DolRecomp` — a real git submodule (own repo, own remote:
  `https://github.com/ExpansionPak/DolRecomp.git`)
- `lib/ModernGekko` — a real git submodule (own repo, own remote:
  `https://github.com/ExpansionPak/ModernGekko.git`), **also has its own
  `native-render-pipeline` branch**
- `lib/ModernGekko/vendor/dolphin` — a nested submodule *inside*
  `lib/ModernGekko` (a Dolphin fork), **also has its own
  `native-render-pipeline` branch**

**When you edit a file, check which repo's working tree it actually lives
in before running `git add`/`git commit` — it is almost never the root repo.**
The established pattern this whole session: commit in the innermost repo
first (e.g. `vendor/dolphin` if you touched `GXRuntime/`), then `git add` +
commit the submodule pointer bump one level up (`lib/ModernGekko`), then one
more level up (root project), each with its own commit message. `git log
--oneline -10` in each of the three repos before committing anything, to see
the established message style (`Phase N: ...` for ModernGekko/dolphin,
`Bump lib/ModernGekko submodule: Phase N ...` for root).

There are actually **two separate copies of DolRecomp** in this tree: the
top-level `lib/DolRecomp` clone (newer emitter, NOT what's actually used) and
`lib/ModernGekko/vendor/dolphin/DolRecomp` (an older, nested-submodule
version — **this is the one the build pipeline actually invokes and links**).
If you're chasing a DolRecomp-codegen question, look in the nested one.

## Environment gotchas (cost real time to discover)

- **The Bash tool cannot invoke `cmd.exe` properly** — it just prints the
  Windows version banner and returns without running anything. Use the
  **PowerShell tool** for any `cmd.exe`/batch-script invocation.
- **cmd.exe's `/c` has a quoting bug**: it only preserves quotes verbatim
  when there are exactly 2 quote characters on the whole command line;
  otherwise it strips only the very first and very last quote char of the
  entire line (not matched pairs), corrupting multi-quoted-argument commands
  built via `std::system()`. Fixed in `moderngekko_port.cpp`'s `RunCommand()`
  by wrapping the whole command string in an extra outer pair of quotes on
  `_WIN32` — if you see mysterious "nombre de archivo... no son correctos"
  errors from `moderngekko-port.exe`, this is why, and it's already fixed;
  don't reintroduce the bug by bypassing `RunCommand()`.
- **GCC/MinGW cannot build ModernGekko** (many Windows-API portability
  errors) — only DolRecomp itself builds fine with GCC. ModernGekko requires
  real MSVC (Visual Studio 2022) via `vcvars64.bat`.
- **MSYS2/mingw64 `pkg-config` on `PATH` breaks CMake's dependency detection**
  when building ModernGekko under MSVC (it finds MinGW-built zlib etc. that's
  ABI-incompatible with MSVC). Every MSVC build script in this project strips
  `C:\msys64\ucrt64\bin`, `C:\msys64\usr\bin`, `C:\msys64\mingw64\bin` from
  `PATH` before running cmake — see `scratch_build_mg2.bat`.
- **Background process discipline**: long builds/runs exceed the 120s tool
  timeout and get silently backgrounded if you're not careful, leading to
  orphaned/duplicate processes. The pattern used throughout this session:
  `PowerShell`'s `Start-Process -PassThru` (captures a real PID) with
  `-RedirectStandardOutput`/`-RedirectStandardError` to log files, then poll
  the PID from Bash (`until ! tasklist //FI "PID eq $pid" ... ; do sleep N; done`,
  run via `run_in_background: true`) rather than blocking. Always check
  `Get-Process -Name moderngekko-run,moderngekko-port,cmd` for leftover
  processes before deleting log files (they hold file locks) or before
  assuming a previous run actually finished.
- **`clangd`/editor diagnostics on these files are mostly false positives** —
  they lack the real MSVC include paths (`std::string`/`std::optional`
  "not found" errors, `moderngekko` namespace "undeclared", etc. on files
  that build and run fine under the real MSVC+Ninja build). Don't chase
  these; trust the actual `cmake --build` output instead.
- **"No D3D12 validation errors" is NOT proof of correct rendering.** This
  bit us at least twice (Phase 2c's degenerate-matrix bug, Phase 3's
  cpixelcenter bug) — both produced completely clean D3D12 debug-layer output
  while rendering nothing/wrong. The only real verification method in this
  project is the framebuffer readback (see below).

## Key build commands

All via the PowerShell tool, from `D:\Proyectos\bt3recompiled`:

```powershell
# Full ModernGekko rebuild (reads scratch_build_mg2.bat)
$p = Start-Process -FilePath "D:\Proyectos\bt3recompiled\scratch_build_mg2.bat" `
  -RedirectStandardOutput "build.log" -RedirectStandardError "build_err.log" -PassThru -WindowStyle Hidden
# then poll $p.Id via tasklist from Bash, or Wait-Process

# Rebuild just the native-render-window probe target (faster iteration)
# reads scratch_build_probe_target.bat, which runs:
#   cmake --build lib\ModernGekko\build --target moderngekko_native_render_window -j 4
```

Headless game boot (regression check — run this after ANY change to
`lib/ModernGekko`, even seemingly-unrelated ones, since several probes share
`dolphin_runtime.cpp`/the `moderngekko` static lib target):

```powershell
cmd.exe /c ""D:\Proyectos\bt3recompiled\lib\ModernGekko\build\moderngekko-run.exe" --game "D:\Proyectos\bt3recompiled\extracted\BudokaiTenkaichi3" --module "D:\Proyectos\bt3recompiled\build\modules\RDSE70\247dff47ae9c0deed3fdddf34d73ceec5e995980d5a14aaff4a3ab710b2ffdbc-5e2bb87324380f04\gRDSE70_recomp.dll" --headless"
```

Expected output (exactly this, every time — if it changes, you regressed
something): `[staticrecomp] core init` then `[staticrecomp] module loaded: ...`.

Windowed game (the actual playable game, not the native-render probe):

```powershell
lib\ModernGekko\build\moderngekko-port.exe run "extracted\BudokaiTenkaichi3" --output "build\modules"
```

Native-render-window probe (the experimental D3D12 pipeline, NOT the real
game):

```powershell
$env:MODERNGEKKO_REAL_GEOMETRY_DUMP = "D:\Proyectos\bt3recompiled\gx_vertex_dump.txt"
lib\ModernGekko\build\moderngekko_native_render_window.exe
```

Capturing fresh GX gameplay data (opt-in env vars, read at process start,
wired in `dolphin_runtime.cpp` via `MaybeEnableGxFifoLogging()`/
`MaybeEnableGxVertexDump()` — both purely observational, never alter
rendering, safe to leave unset):

```powershell
$env:MODERNGEKKO_GX_LOG = "D:\...\path.log"              # raw command/state text log (Phase 0)
$env:MODERNGEKKO_GX_VERTEX_DUMP = "D:\...\path.txt"       # decoded-geometry + texture dump (Phase 2b-4)
lib\ModernGekko\build\moderngekko-port.exe run "extracted\BudokaiTenkaichi3" --output "build\modules"
```

## Verifying rendering: the framebuffer-readback method

`tests/native_render_window.cpp` (in `lib/ModernGekko`) writes a `.ppm`
framebuffer dump + prints a pixel-region summary to stdout after presenting a
frame. To actually LOOK at it (since neither the shell tools nor a screen-blind
AI can watch a live window):

```powershell
Add-Type -AssemblyName System.Drawing
$bytes = [System.IO.File]::ReadAllBytes("path.ppm")
$headerBytes = [System.Text.Encoding]::ASCII.GetBytes("P6`n640 480`n255`n")  # confirm actual header first!
$offset = $headerBytes.Length
$bmp = New-Object System.Drawing.Bitmap 640, 480
for ($y = 0; $y -lt 480; $y++) { for ($x = 0; $x -lt 640; $x++) {
  $i = $offset + ($y * 640 + $x) * 3
  $bmp.SetPixel($x, $y, [System.Drawing.Color]::FromArgb($bytes[$i], $bytes[$i+1], $bytes[$i+2]))
} }
$bmp.Save("out.png", [System.Drawing.Imaging.ImageFormat]::Png)
```

Then use the Read tool on the resulting PNG — Claude Code can view images
directly. This is how every "it renders correctly" claim in this project's
history was actually confirmed, not assumed from clean logs.

Existing evidence files at the project root (kept intentionally, don't
delete): `phase2c_frame0_readback.png`, `frame0_readback_multidraw.png`,
`phase3c_frame0_readback.png`, `phase4_frame0_readback.png`,
`gx_real_texture2.png`, `gx_gameplay.log` (66 MB raw capture, has a
`# --- summary ---` block at the end — never `cat`/`Read` the whole thing,
use `grep`/`head`/`tail`).

## Current state of the native-render-pipeline work

Read `README.md`'s "Native rendering pipeline" section for the user-facing
phase summary. Technical detail an implementer needs:

- **Real shader**: `moderngekko::DolphinShaderCompiler::Compile()`
  (`lib/ModernGekko/src/gpu/dolphin_shader_compiler.cpp`) wraps Dolphin's real
  `PixelShaderGen`/`VertexShaderGen`/`UberShader*` (vendored under
  `vendor/dolphin_legacy/VideoCommon/`, a SEPARATE, older Dolphin copy from
  `vendor/dolphin/` — don't confuse the two). Feed it real captured CP/XF/BP
  register arrays (see `CompileFromCapturedState()` in
  `tests/native_render_window.cpp` for the exact captured values from a real
  BT3 combat: `GENMODE=0x4001`, `TREF=0x49040`, `BLENDMODE=0x4a0`,
  `TEV_COLOR/ALPHA_ENV=0x8fff8/0x8ffc0`).
- **GLSL→HLSL**: `moderngekko::TranslateGlslToHlsl()`
  (`include/moderngekko/glsl_to_hlsl.hpp`, `src/gpu/glsl_to_hlsl.cpp`) is a
  standalone target (`moderngekko_glsl_to_hlsl`) wrapping glslang
  (GLSL→SPIR-V) + `spirv_cross::CompilerHLSL` (SPIR-V→HLSL), built with ZERO
  dependency on Dolphin's Common/VideoCommon libs. This was deliberately kept
  separate to avoid an ODR (One Definition Rule) violation: Dolphin's real
  D3D backend code (`vendor/dolphin/Source/Core/VideoBackends/D3DCommon/`)
  and the `vendor/dolphin_legacy`-based shadergen target both define globals
  like `g_ActiveConfig`/`MsgAlertFmtImpl`/`File::Exists` — linking both into
  one binary causes LNK2005 duplicate-symbol errors. If you need to touch
  shader translation, work through `glsl_to_hlsl.cpp`, don't try to call
  Dolphin's real `D3DCommon::Shader::CompileShader` directly.
- **glslang's own resource-limits shim** needed
  `_CRT_SECURE_NO_WARNINGS` added (scoped to just the
  `glslang-default-resource-limits` CMake target) because MSVC's `/WX`
  (warnings-as-errors) turns a `strncpy` deprecation warning fatal — see the
  `CMakeLists.txt` comment near that target.
- **The render probe**: `lib/ModernGekko/tests/native_render_window.cpp`
  (target `moderngekko_native_render_window`) is a standalone, from-scratch
  D3D12 device/swapchain/window — it does NOT use any of ModernGekko's own
  `Platform`/`AbstractGfx` abstractions, deliberately, to keep it simple and
  isolated from the real game's render path. It:
  1. Compiles the real captured shader via the two systems above,
     `D3DReflect`s the bytecode to build the root signature + input layout +
     descriptor heaps generically (doesn't hardcode resource bindings),
  2. Loads real decoded geometry via `LoadRealGeometry()` (reads a
     `gx_vertex_dump.txt`-style capture) instead of a hand-picked triangle,
     normalizing raw GX vertex-space coordinates to NDC via one shared
     bounding box across all captured draws,
  3. Fills unknown cbuffer variables via `FillIdentityAndOnes()` — a
     **name-based heuristic**, not real reflection-type detection, because
     spirv_cross flattens GLSL `mat4` uniforms into plain
     `D3D_SVC_VECTOR` entries with no matrix class in D3D12 shader
     reflection. Known real Dolphin uniform names handled specially:
     `cpnmtx`/`cproj`/`ctexmtx`/`ctrmtx`/`cnmtx`/`cpostmtx` (identity-filled)
     and **`cpixelcenter`** (a genuine D3D half-texel correction — this one
     bit us badly, see "Two bugs that cost the most time" below; it needs
     x>0/y<0/z=-1/w=0, NOT identity, NOT the generic 1.0f fill).
  4. Uploads a texture — real decoded C8+TLUT game texture if
     `LoadRealTexture()` finds a dump, else a synthetic 4-quadrant
     red/green/blue/yellow fallback (upgraded from an earlier plain
     checkerboard, for easier UV-orientation debugging via readback).
  5. Issues one `DrawIndexedInstanced` per captured draw, presents, and
     (critically) reads back the actual rendered backbuffer to a `.ppm` +
     stdout pixel summary — this is the verification method, not "no errors".
- **Texture/TLUT capture**: `GxVertexDumpDevice`
  (`include/moderngekko/gx_vertex_dump.hpp`, `src/gpu/gx_vertex_dump.cpp`) is
  an opt-in `GxRenderDevice` (via `MODERNGEKKO_GX_VERTEX_DUMP=<path>`) that:
  - Snapshots real Wii RAM (`AddressSpace`, refreshed from
    `Core::System::GetInstance().GetMemory()` each FIFO batch) so
    `GxCommandProcessor`/`GxStateBackend` can resolve display-list contents
    and indexed vertex-array reads against real, live game data — **this
    was missing entirely until Phase 3**, meaning nothing before Phase 3
    actually decoded display-list contents (BT3 renders almost everything
    via display lists per Phase 0's capture: 20,773 distinct addresses).
  - Dumps the first N (16, as of Phase 3) real decoded draw calls
    (≥3 vertices) as plain text (position/UV/vertex-color per vertex,
    indices), merged by `LoadRealGeometry()` in the probe.
  - Resolves the real bound texture's TLUT (palette) by reading Dolphin's
    real, live `s_tex_mem` global (`VideoCommon/TextureDecoder.h`) at the
    offset/format given by BP register `BPMEM_TX_SETTLUT` (0x98, unit 0) —
    TLUT lives in TMEM (a separate 1MB region from main RAM), populated by
    the real running video backend's own BP handler, so this reads that
    backend's own state rather than re-modeling TMEM loads. There's a real
    race (video thread populates TMEM asynchronously vs. when the raw-FIFO
    observer runs on the CPU thread) handled by retrying on a later draw if
    the palette reads as all-zero.
  - Real GX texture decoding itself (C4/C8/C14X2/etc. tiled-block formats)
    was ALREADY fully implemented in `src/gpu/gx_texture_decoder.cpp`
    (`GxTextureDecoder::Decode()`) before this session touched it — the gap
    was entirely in *acquiring* real palette bytes, not decoding them.

### Two bugs that cost the most time (read before touching cbuffer fill logic)

1. **Degenerate identity-matrix fill** (Phase 2c): the original
   `FillIdentityAndOnes()` only detected matrices via D3D12 reflection's
   `D3D_SVC_MATRIX_ROWS`/`D3D_SVC_MATRIX_COLUMNS` class — but spirv_cross
   never produces that class for cross-compiled GLSL `mat4` uniforms, so
   ZERO of Dolphin's real transform matrices were ever identity-filled; they
   all got the generic 1.0f pattern instead, which is a degenerate,
   non-invertible "matrix" that collapses every vertex to roughly the same
   point off-screen. Fixed with the name-based heuristic described above.
2. **`cpixelcenter` corrupting clip-space position** (Phase 3c, the actual
   root cause of what Phase 3 first misdiagnosed as a "clip-space convention"
   problem): even after fixing #1, one specific uniform — `cpixelcenter`, a
   genuine D3D pixel-center/half-texel correction Dolphin's real shadergen
   applies via `o.pos.xy -= cpixelcenter.xy * o.pos.w` — was still getting
   the generic 1.0f treatment (it's a plain vec4, not "matrix-like" by any
   heuristic), which turned that shader line into a constant (-1,-1)
   clip-space translation, pinning ALL rendered geometry into the
   bottom-left corner regardless of its actual size/position. This is why
   Phase 3's 4x-larger multi-draw capture showed *no visible size change* —
   the bug wasn't about how much geometry was captured, it was a constant
   post-transform offset applied to everything. **If you see geometry stuck
   in a screen corner again, check every uniform name-matched in
   `FillIdentityAndOnes`/`ReflectResources`, not just the obvious "matrix"
   ones** — Dolphin's real shadergen has several small special-purpose
   uniforms like this (look at `VertexShaderGen.cpp` for anything written
   into `o.pos` besides the obvious model/view/projection multiply).

## What's NOT done yet / next steps

1. **Real combat texture capture.** The one real C8+TLUT texture actually
   resolved (Phase 4) came from an automated ~60s headless capture and
   happened to be near-black/low-alpha — almost certainly a UI/menu element,
   not character/effect art. To get something visually interesting, repeat
   what worked in Phase 2b: have a human actually play a real combat with
   `MODERNGEKKO_GX_VERTEX_DUMP` set (interactive windowed session, not
   automated headless), then re-run the texture-resolution capture against
   that session.
2. **More than a handful of merged draw calls.** Currently 16 draws get
   merged into one scene fragment (a "UI tile mosaic" per Phase 3's
   description) using one shared bounding box. Rendering toward an actual
   full assembled frame needs a real camera/projection setup (current code
   explicitly avoids building one — everything is normalized via bounding
   box, which only works for near-planar/UI-style geometry, not real 3D
   character meshes with actual depth).
3. **TEV combiner coverage beyond what's been captured.** Only the specific
   TEV/BP state captured in the reference combat session has been proven to
   shader-gen and render correctly. Phase 0's capture showed 500+ distinct
   TEV_COLOR_RA states across a real combat — most haven't been individually
   verified through the full pipeline yet.
4. **Eventually**: retire the Dolphin GX/VideoCommon dependency for
   real-game rendering (not just this standalone probe) once feature parity
   is reached — that's the actual "no more GPU emulation" milestone the
   whole effort is aimed at. Currently the real, playable game still renders
   through the full inherited Dolphin stack; the native-render-window probe
   is a parallel, disconnected proof-of-concept, not yet wired into
   `moderngekko-run.exe`'s actual boot/render path.

## Scratch files at project root

`scratch_*.bat`/`.ps1` files are reusable build/run wrappers accumulated
across this session (not committed — gitignored via `scratch_*.bat` pattern).
Reuse them rather than reinventing the vcvars64/PATH-stripping dance each
time. If you delete or modify one, there is a history of forked sub-agents
accidentally destroying these and needing to recreate them from memory —
don't be the next one; check `ls scratch_*.bat` before any broad cleanup and
restore anything you remove that's still referenced above.

`*.log`/`*.pid` files at the root are gitignored scratch build/run output —
safe to delete freely EXCEPT `gx_gameplay.log` (a real data capture, not a
build log, despite the extension) and the `phaseN_*.png`/`.ppm` readback
evidence files (real verification artifacts, not disposable logs).

A few untracked stray files (`real_vs.hlsl`, `real_ps.hlsl`,
`real_vs_debug.hlsl`, `real_ps_debug.hlsl`, `scratch_launch_build.ps1`) exist
at the root from an unidentified earlier process — several sessions in a row
have noted these as "not mine, leave alone", so do the same unless you
specifically need them and can determine they're safe to touch.
