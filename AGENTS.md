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
