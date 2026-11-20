# Project memory — PS4 homebrew

Everything learned building the Snake game in `ps4/`. Read this before starting
any PS4 homebrew work here: the two bugs below both build cleanly and only fail
on real hardware, so nothing in the toolchain warns you.

## Toolchain

`/home/openhands/ps4/PS4TC` (`OO_PS4_TOOLCHAIN`). Required environment:

```sh
export PATH=/home/openhands/bin:$PATH
export OO_PS4_TOOLCHAIN=/home/openhands/ps4/PS4TC
export LD_LIBRARY_PATH=/home/openhands/libssl11/usr/lib/x86_64-linux-gnu
export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1
```

`PkgTool.Core` is a .NET binary and fails without the last two.

## Bug 1 — malformed TITLE_ID

`TITLE_ID` must be exactly **4 uppercase letters then 5 digits**: `SNAK00001`.
Five letters and four digits (`SNAKE0001`) looks more natural but the console
rejects the package at install with **`0x80990055`** (`-2137456555` signed).
The build succeeds, `pkg_validate` passes, and the failure appears only when the
user tries to install. Every OpenOrbis sample uses the 4+5 form
(`BREW00083`, `BREW00052`, `BREW00084`).

`CONTENT_ID` is exactly 36 chars: `IV0000-<TITLE_ID>_00-<16 chars>`. The 16-char
tail must not repeat the title id — `SNAKEPS400000000`, not `SNAKE000010000000`.

`snake/Makefile` asserts both with `$(error ...)`, so a malformed id now fails the
build instead of the install. Keep that guard in any new project.

## Bug 2 — the splash screen never hidden

**This is the dangerous one.** Without the hide call the launcher's splash image
stays drawn on top of the application permanently. The app is fully alive
underneath: input is processed, frames render, flips succeed, audio plays — the
user just sees one frozen image.

Symptom as users describe it: *"the game is stuck on the launch image, but
pressing the PS button shows it running for a moment, and my moves are applied
when I come back — like the game runs behind the image."* Treat that wording as
a positive diagnosis, not as a rendering bug.

```cpp
sceSysmoduleLoadModuleInternal(ORBIS_SYSMODULE_INTERNAL_SYSTEM_SERVICE);
sceSystemServiceHideSplashScreen();
```

The internal sysmodule load must come first or the hide call fails. Needs
`#include <orbis/Sysmodule.h>` and `-lSceSysmodule`. Reference implementation:
`samples/piglet/piglet/PigletApplication.cpp`. Call it right after opening video,
before the first frame.

Do not chase this as a double-buffering bug. Rendering into `buffers[current]`
and then flipping that same buffer is correct; a wrong buffer would show torn or
stale frames that still update, not one frozen image.

## Verifying a package

`pkg_validate` checks hashes and signatures only — it does **not** catch a
malformed title id, so validate and inspect separately:

```sh
PkgTool.Core pkg_validate <file>.pkg      # expect all [OK], zero failures
PkgTool.Core pkg_listentries <file>.pkg   # PARAM_SFO sits in the header
dd if=<file>.pkg of=/tmp/p.sfo bs=1 skip=$((0x3990)) count=$((0x434))
PkgTool.Core sfo_listentries /tmp/p.sfo   # confirm TITLE_ID / CONTENT_ID
```

Traps worth remembering:

- `PARAM_SFO` is a package **header** entry, not a file inside the PFS image. An
  extracted `uroot/` legitimately has no `param.sfo`. Do not report it missing.
- `strings` on a `.pkg` finds nothing, because `eboot.bin` sits inside a
  compressed PFS. Run `pkg_extract` first, then `strings` the extracted file.
- `SYSTEM_VER = 0`, `CATEGORY = gd`, `APP_TYPE = 1`, `ATTRIBUTE = 0` match the
  official samples exactly. They are correct, not defects.
- Comparing against a sample that is known to work is the fastest way to rule
  out a packaging problem: build `samples/hello_world`, then diff the headers
  field by field and diff the extracted file lists. Legitimate differences are
  confined to hash/signature regions.
- A package the same byte size as a working sample is strong evidence the
  container is fine and the defect is in the metadata or the code.

## Sharing a build with the user

Serve from `/tmp/dl` with `python3 -m http.server 12000 --bind 0.0.0.0` and hand
out `https://work-1-<session>.prod-runtime.all-hands.dev/<file>.pkg`. The
`work-2` host is not routed (502). Always verify by downloading the file over the
public URL and comparing `sha256sum` with the local build, and check that a fix
is actually present in the *downloaded* artifact by extracting it. The server
dies when the session ends, so restart it on demand.

## Rebuild hygiene

`make` does not track `param.sfo` correctly. After changing the id, the
toolchain, or anything in `sce_sys/`:

```sh
rm -f *.pkg pkg.gp4 eboot.bin sce_sys/param.sfo && rm -rf build
```

Stale `.pkg` files from an old id linger otherwise, and it is easy to ship the
wrong one. Grep the repo for the old id after renaming (`README.md`, `build.sh`
and docs are easy to miss).

## Notes on this codebase

`snake/src/game.cpp` is the shared game core and compiles for both the console
and a desktop harness via `-DSNAKE_PREVIEW`. `snake/src/main.cpp` is the
PS4-only layer: VideoOut double buffering, pad polling, procedural audio through
AudioOut, lifecycle. `ps4/build.sh` regenerates assets, packs the blob, runs the
harness, then builds the package — run it after touching assets or game logic.

`tools/gen_assets.py` pre-renders Arabic into white alpha PNG sprites at build
time, tinted at draw time, because PS4 gives homebrew no Arabic shaping (no
harfbuzz). It relies on Pillow's `raqm` support for `direction="rtl"`; check
`PIL.features.check("raqm")` before blaming the font. `arabic_reshaper` and
`python-bidi` are **not** used and are not installed. `tools/pack_assets.py`
bakes 48 sprites into `snake/assets/snake.raw` (~2.3 MB): magic `SNAK`, version
1, count, then per sprite `width, height, 0xAARRGGBB pixels[]`. Sprite order must
match the `SpriteId` enum in `snake/src/game.h`.

Platform details that are easy to get wrong:

- Video: 1920x1080, two buffers, `sceVideoOutSetBufferAttribute(&attr,
  0x80000000, 1, 0, w, h, w)` — the leading `0x80000000` is the format flag and
  the last argument is the pitch in pixels.
- Draw into `video.buffers[video.current]`, then submit that same index.
- `sceLibcHeapSize = 64 MB` in `src/main.cpp` covers the 8 MB backdrop plus the
  2 MB blob. It must be a global named exactly `sceLibcHeapSize`.
- Homebrew cannot assume a writable mount. The high-score save tries `/data`,
  `/temp0`, `/app0` in order and remembers whichever accepts the write.
- Stick axes arrive as 0..255 with 128 at rest; deadzone applied after scaling.
- A missing controller must not freeze the loop — keep rendering the menu.
- Write the lightbar once, not every frame.

## Working style that mattered here

The user reported two bugs and both times the decisive clue was their own
description, not the code. Ask what exactly they see before reading the source,
and take an unusual report ("it runs behind the image") literally — it usually
names the mechanism. Also state plainly when something cannot be verified
without the console instead of implying a fix is confirmed.
# Workspace notes

## Projects

- `AutoPRX/` â€” GoldHEN plugin loader with a second config file
  (`/data/GoldHEN/AutoPRX.ini`) alongside `plugins.ini`. Working on hardware.
- `PlayerList/` â€” in-game menu skeleton for GoldHEN. Builds and loads; menu
  actions and player list are intentionally unimplemented (need game offsets).

## Build environment

```sh
export OO_PS4_TOOLCHAIN=/tmp/ps4/OpenOrbis/PS4Toolchain
export GOLDHEN_SDK=/tmp/gh_sdk
export PATH="$OO_PS4_TOOLCHAIN/bin/linux:$PATH"
```

## GoldHEN plugin build facts (verified)

- `create-fself` needs `--lib=`, not `--out=`:
  `create-fself -in=x.elf -out=x.elf.oelf --lib=x.prx --paid 0x3800000000000011`
  Run it with a cwd-relative `-in` path or the PRX embeds the absolute path and
  builds stop being reproducible.
- Thread creation is `scePthreadAttrSetdetachstate` (lowercase `d`), and the
  thread function must be `void *(*)(void *)`.
- `sceKernelGetProcessTime()` returns monotonic microseconds â€” use it for
  timing instead of `sceKernelGettimeofday`.
- `klog` is `#define klog printf` in `Utilities.h`; `plugin_common.h` in both
  projects wires it up.
- Plugin exports needed for both loaders: `module_start`, `module_stop`,
  `plugin_load`, `plugin_unload`, plus `g_pluginName/Desc/Auth/Version`.
- Do **not** call `scePadInit` from a plugin â€” the game already owns the pad.
  Use `scePadGetHandle(0, 0, 0)` + `scePadReadState`.
- `-lSceLibcInternal -lGoldHEN_Hook -lkernel -lSceSysmodule -lScePad -lSceSysUtil`
  links successfully and the result loads on hardware.

## What cannot be done without the game binary

Drawing an overlay needs a hook on the game's present/render function, and
reading game state needs validated offsets. Both are build-specific. Do not
invent offsets or native hashes â€” a wrong hash calls an arbitrary address
instead of failing cleanly. Keep such tables in one file with a `build_id` and
a `valid` flag so a stale table is detected rather than dereferenced.

## Testing pattern

Platform-independent logic (menu state machine, parsers) lives apart from
Orbis calls so it compiles with the host compiler. `make host-test` runs it.
Mutation-test the suite before trusting it: flip a comparison and confirm a
failure appears.

---

# AutoPRX

## Overview

GoldHEN plugin that loads PRX files per game Title ID from
`/data/GoldHEN/AutoPRX.ini`, alongside GoldHEN's own `plugins.ini`. Logs under
the `[AutoPRX]` prefix.

| Path | Role |
| --- | --- |
| `src/main.cpp` | Entry points, worker thread, `plugin_load`/`module_start` |
| `src/prx_loader.cpp` | Load engine, duplicate detection, error diagnostics |
| `src/game_detector.cpp` | Title ID detection from the running process |
| `src/config_parser.cpp` | INI parser for `AutoPRX.ini` |
| `src/logger.cpp` | Append-only log to `/data/GoldHEN/AutoPRX.log` |
| `src/plugin_common.c` | GoldHEN SDK glue |
| `include/autoprx.h` | Shared constants and limits |
| `payload/autoprx_inject.c` | Freestanding injector (no libc), polls for `eboot.bin` |
| `tools/inject.py` | Host-side payload tool |
| `tools/make_fself.py` | Alternative SELF wrapper |
| `tests/` | Host tests: `autoprx_tests`, `payload_tests` |
| `examples/` | Reference `AutoPRX.ini` and `plugins.ini` |
| `Makefile` | OpenOrbis toolchain, `create-fself` |
| `.github/workflows/build.yml` | CI |

Host tests currently pass: `autoprx_tests` 73 checks, `payload_tests` 49
checks, 0 failures.

## Code issues from review

### [OPEN] argv dangling pointer
`src/main.cpp:56-59` declares `WorkerArg { const char *trigger; int argc;
char **argv; }`, and `autoprx_start()` stores the caller's pointers at
`:278-279` without copying. `worker_thread()` sleeps
`AUTOPRX_START_DELAY_US` (250 ms, `include/autoprx.h:25`) at `:78` before
using them. `plugin_load`'s argv belongs to GoldHEN's frame and may be gone by
then. No `strcpy`/`strncpy`/`memcpy` of argv exists anywhere in `src/`.
Fix: copy argc and the argv strings into static storage before `plugin_load`
returns.

### [OPEN] sceKernelGetModuleList size argument (unverified)
`src/prx_loader.cpp:175` calls
`sceKernelGetModuleList(handles, sizeof(handles), &available)` â€” 1024 bytes for
a 256-element array. The second parameter is likely the element count (256),
not bytes. The OpenOrbis header
(`include/orbis/libkernel.h:203`) is declared
`sceKernelGetModuleList(OrbisKernelModule *array, size_t size, size_t *available)`
with an empty comment and no parameter names, so it does not settle the
question. **Unverified** â€” needs a runtime check. Note the clamp at `:177`
already assumes a count.

### [OPEN] path_basename called before the NULL check
`src/prx_loader.cpp:368` initialises
`const char *module_name = path_basename(path);` at declaration time, while the
`if (!path || path[0] == '\0')` guard sits at `:371`. `path_basename` does
`strrchr(path, '/')` and dereferences NULL. Move the guard above the
initialiser.

### [OPEN] Workflow paths vs repo layout
`.github/workflows/build.yml` lives at `AutoPRX/.github/`, but the git root is
the parent directory. GitHub only reads workflows from the repo root, so this
one is never picked up. It also uses `working-directory: AutoPRX/tests` and
`working-directory: AutoPRX`. Either move `.github/` to the repo root and drop
the `AutoPRX/` prefix, or make `AutoPRX/` the repo root consistently.

### [OPEN] README contradicts the header check
`README.md:186-188` says the payload "checks that `AutoPRX.prx` exists, is
readable, and starts with an ELF header before doing anything else, and stops
with an error if it does not." The code does the opposite:
`payload/autoprx_inject.c:518-524` documents the checks as advisory, and an
unreadable file only logs
`warning: ... is not readable from the payload context, trying the load anyway`.
Update the README to match.

### [OPEN] g_syscall_errno defined twice
`payload/autoprx_inject.c:97` (outside the conditional) and `:120` (inside the
`#else`) both declare `static long g_syscall_errno;`. Remove one.

### [OPEN] Prebuilt binaries not fully ignored
`.gitignore` lists `bin/*.elf` and `bin/*.oelf` but **not** `bin/*.prx`, so
`bin/AutoPRX.prx` is committable. It also lists `tests/autoprx_tests` but not
`tests/payload_tests`. Build both in CI and publish as release artifacts
instead of committing them.

### [OPEN] Benign payload log noise
Under Launchpad `read()` returns 0, so the payload prints
`prx header: file too short (0 bytes read)` and
`read() self-check on the log: 0 bytes, errno 0 (read() itself is failing)`
(`payload/autoprx_inject.c:409-447`). The PRX still loads â€” confirmed by the
`loaded ... into eboot.bin` line that follows. Consider not printing these as
errors.

## Runtime finding: 0x80020008 is a packaging problem, not an AutoPRX bug

Symptom reported from hardware, game `CUSA08519` v01.32:

```
[AutoPRX] ERROR sceKernelLoadStartModule failed 0x80020008 for
          /data/GoldHEN/AutoPRX/Sheriff-132v106.prx
Result: 0 loaded, 1 failed
```

- `0x80020008` = `0x80020000` + errno 8 = **ENOEXEC**: the kernel opened the
  file, read the header, and rejected it.
- Diagnostics from `log_module_header()` (`src/prx_loader.cpp:104`): size
  434952 bytes, plain ELF, class=2, data=1, type=0xfe18, machine=0x3e,
  phnum=7, entry=0.
- `0xfe18` is `ET_SCE_EXEC`; a loadable PRX must start with the SELF magic
  `4f153d1d`, not `7f454c46`. The file was an unwrapped ELF/oelf.
- Both files are still in the workspace and confirm this:
  `Sheriff-132v106.prx` = 434952 bytes, magic `7f454c46`, type 0xfe18;
  `Sheriff-132v106-self.prx` = 389808 bytes, magic `4f153d1d`.

Fix path â€” wrap it, then use the `--lib` output:

```sh
create-fself -in=<input> -out=<name>.oelf --lib=<name>.prx --paid 0x3800000000000011
```

Use the file named by `--lib` (the final SELF), **not** `-out` (the
intermediate oelf). Verify with `xxd <file> | head -1` â†’ must start with
`4f15 3d1d`, and the size must differ from the unwrapped input.

- **Unverified:** whether `create-fself` rejects an input that is already an
  oelf (type 0xfe18), and whether `tools/make_fself.py` is a workable
  alternative. Neither was tested.
- Discriminating test: put the same path in `plugins.ini` under `[CUSA08519]`
  and let GoldHEN's own loader handle it. If that also fails, the file is the
  problem, not AutoPRX.
- The second run reported the same 434952 bytes, meaning the file had not
  actually been replaced on the device yet.

Note: the `AutoPRX.log` currently in the workspace root is a **later** run and
no longer contains `0x80020008`; it shows the wrapped file loading.

## Diagnostics to keep

In `src/prx_loader.cpp` `log_module_header()` (`:104`): file size, readable
byte count, ELF class/data/type/machine, and entry/phoff/phnum/phentsize/ehsize.
Two error codes get named: `0x80020002` â†’ "plugin not found" and `0x80020008`
â†’ ENOEXEC, which adds the unwrapped-SELF hint. Keep the wording that the
loadable `.prx` is the `--lib` output of `create-fself`.

In `payload/autoprx_inject.c`: SDK version probe, PRX header hex dump,
`lseek`-based size (works when `read()` is broken), and a `read()` self-check
on the log file.

## Debugging checklist

1. Read the `[AutoPRX]` lines in order: `Detected Game:`, then
   `Configuration loaded: ok (lines= sections= entries= disabled= duplicates=
   invalid= orphan= overflow=)`, then the section lookup, then the load result.
2. `0x8002xxxx` â†’ errno is the low byte.
3. When a load fails, check the magic bytes and the file size before blaming
   the loader. SELF `4f153d1d` is loadable; ELF `7f454c46` with type 0xfe18 is
   not.
