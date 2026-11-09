# RESOGUN for PC (resogun-recomp)

A native PC port of **RESOGUN** (Housemarque, PlayStation 4) for Windows and Linux / SteamOS,
built on the [wipeout_pc](https://github.com/AbdelmoumenDeghbouche/wipeout_pc) runtime: the
game's original x86-64 code runs directly on the PC's processor, a runtime replaces the PS4
system libraries, and the graphics are translated to Vulkan by the shadPS4 video core. There is
no CPU emulator.

> **Status: just started. Not playable yet.** The runtime is imported unchanged from wipeout_pc
> and a RESOGUN profile is added. Next: list and implement the missing PS4 imports.
> Analysis notes: [docs/resogun.md](docs/resogun.md).

No game files are included. You need your own dump of the game.

## Supported game versions

| Title ID | Region | Version | State |
|---|---|---|---|
| CUSA00038 | US | 01.00 + update 01.10 | Development target |

The base game and the update are merged into one folder (the update's files replace the base
game's). The intro videos only come with the base game.

## Building (Windows)

1. Install [MSYS2](https://www.msys2.org) (`winget install MSYS2.MSYS2`) and, in the
   **MSYS2 CLANG64** shell, the packages listed in
   [packaging/windows/README.md](packaging/windows/README.md).
2. Install Python 3.10+ from python.org (with Tk), then:

   ```powershell
   python -m pip install pillow iced-x86==1.21.0 cryptography
   ```

3. Clone with submodules and build in the MSYS2 CLANG64 shell:

   ```bash
   git clone --recursive https://github.com/darkuni/resogun-recomp
   cd resogun-recomp
   bash build.sh
   ```

   If Git converts line endings on your PC (`core.autocrlf=true`), the FSR patches do not
   apply. Turn it off for this repository and its submodules before building:

   ```bash
   git config core.autocrlf false
   git submodule foreach --recursive git config core.autocrlf false
   git rm -rq --cached . && git reset -q --hard
   git submodule foreach --recursive 'git rm -rq --cached . && git reset -q --hard'
   ```

## Preparing your game

From fake PKGs of your own dump (base game and update):

```powershell
python scripts/pkg_extract.py <base.pkg> D:\games\CUSA00038
python scripts/pkg_extract.py <update.pkg> D:\games\CUSA00038-patch
```

Extraction first reads the PKG to calculate its SHA-256. A retry keeps only files recorded as
complete for that exact package whose size and modification time have not changed. Older
extractions without this record are extracted again once, even when file sizes match.

Then move the update's files over the base game folder, or pass the update folder with
`GOW3_PATCH_DIR`. A folder dumped with GoldHEN works the same way.

## Running

Start the launcher, choose your game folder on the **Game** page, then click **Play**:

```powershell
python launcher\gow3_launcher_win.py
```

The **Advanced** page can put a shortcut on the desktop. Settings: display mode, frame rate
limit, VSync, anisotropic filtering, HDR, controls, mods and the game language.

Without the launcher, `run.py` starts the game:

```powershell
$env:GOW3_GAME_DIR = 'D:\games\CUSA00038'
python run.py
```

List the PS4 functions the runtime does not have yet:

```powershell
$env:GOW3_GAME_DIR = 'D:\games\CUSA00038'
$env:GOW3_MODS_ENABLED = '0'
python run.py --check-imports
```

Without `--check-imports`, `run.py` starts the game. The loader stops at the first system
function the runtime does not implement and prints its name.

## How it works

| Part | Where | What it does |
|---|---|---|
| PKG unpack | `scripts/pkg_extract.py` | Decrypts and unpacks fake PKGs (PFS, PFSC), as shadPS4 does |
| Game profiles | `scripts/game_profiles.py` | System modules to link per title ID |
| Preparation | `scripts/prepare.py`, `link_libc.py`, `link_modules.py` | Reads `eboot.bin` and links the game's own `libc`, `libSceFios2` (and `libSceNpToolkit2` for WipEout) |
| Import names | `scripts/gen_import_names.py`, `src/import_names.inc` | Maps the game's import NIDs to function names |
| Loader | `src/probe.c` | Loads the image and starts the game's original code |
| Runtime | `src/runtime_*.c` | PS4 system functions |
| Audio3d | `src/runtime_audio.c` | Mixes the bed and the sound objects of the Wwise sink into one output |
| Json2 | `src/runtime_json.c` | sce::Json values, parser and serializer |
| Red zones | `src/redzone.cpp`, `scripts/function_starts.py` | Keeps Windows exception records out of the game's red zones |
| Launcher | `launcher/` | Settings window and game start |
| GPU | `gpu/shadps4/` | shadPS4 video core (Vulkan) |

Many names (`gow3-probe.exe`, `GOW3_*` variables, the launcher) still come from the God of
War III port this runtime started as. WipEout Omega Collection and God of War III profiles stay
in place while the runtime is shared.

## Credits

This project is a fork of AbdelmoumenDeghbouche's WipEout Omega Collection port, which is a fork
of FelipePrado0's God of War III port, which comes from Supermedo's Windows port of
deadinside28's original:

- **AbdelmoumenDeghbouche**: [wipeout_pc](https://github.com/AbdelmoumenDeghbouche/wipeout_pc),
  the base of this project (Audio3d/Wwise mixing, Json2, game profiles, red-zone protection).
- **FelipePrado0**: [SpartanPort](https://github.com/FelipePrado0/SpartanPort), the God of War III
  port that runtime comes from.
- **bc160201983**: [medievil_2019_pc_port](https://github.com/bc160201983/medievil_2019_pc_port),
  the first port of another game on that runtime. `pkg_extract.py`, `game_profiles.py` and
  `gen_import_names.py` come from it.
- **deadinside28**: [bloodborne_pc](https://github.com/deadinside28/bloodborne_pc), the original
  native port.
- **Supermedo**: [bloodborne_pc for Windows](https://github.com/Supermedo/bloodborne_pc).
- **cuesta4**: [shadPS4 fork](https://github.com/cuesta4/shadPS4/tree/eltutz) (branch `eltutz`):
  the GNM blend rewrite, depth/stencil state and image aliasing fixes.
- **shadPS4 team**: the [shadPS4](https://github.com/shadps4-emu/shadPS4) video core, the PKG
  format and the symbol names (`tools/aerolib.inl`).

Contributors to the WipEout port: **d4g** (render resolution patches, the Metro Park terrain fix),
**carlosmarind** (save and heap fixes, PS4 Pro mode work), **Extremenis** (the overexposed SDR
fix), **ahmanwoods** (Audio3d IDs), **alhockly** (EU version test), **benhaslett** (game clock
measurements, Linux build fix) and **iannitram** (PS4 FPU state).

Also used: [LibAtrac9](https://github.com/Thealexbarney/LibAtrac9), [FFmpeg](https://ffmpeg.org),
[SDL3](https://github.com/libsdl-org/SDL), [Dear ImGui](https://github.com/ocornut/imgui),
[sirit](https://github.com/shadps4-emu/sirit), [magic_enum](https://github.com/Neargye/magic_enum),
[miniz](https://github.com/richgel999/miniz), [xbyak](https://github.com/herumi/xbyak),
[Vulkan Memory Allocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator),
[FSR-Vulkan](https://github.com/FireBurn/FSR-Vulkan) and the AMD FidelityFX SDK,
[MSYS2](https://www.msys2.org), [LLVM](https://llvm.org) and [Pillow](https://python-pillow.org).

## License

GNU GPL v2 or later ([LICENSE](LICENSE)). Third-party components keep their own licenses.
