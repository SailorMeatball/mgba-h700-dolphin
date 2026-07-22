# mGBA — H700 Dolphin Link (SDL frontend)

A fork of [mGBA](https://github.com/mgba-emu/mgba) that adds a `--dolphin <ip>` command-line
flag to the **SDL frontend**, exposing the existing (Qt-only) "Connect to Dolphin"
GBA↔GameCube link-cable feature on Allwinner H700-based handhelds (RG34XX, RG35XXH,
CubeXX, etc.) running Buildroot-based custom firmware such as KNULLI or muOS.

The stock firmware on these devices only ships mGBA's lightweight SDL build — no Qt,
no GUI to trigger the Dolphin connection. This patch wires the same core-level
functionality (already fully implemented in `src/gba/sio/dolphin.c`, used by the Qt
"File → Connect to Dolphin" menu) into the SDL frontend via a new CLI flag instead.

## What this actually changes

Three files, no core emulation logic touched:

- `include/mgba/feature/commandline.h` — adds `dolphinAddress` to `struct mArguments`
- `src/feature/commandline.c` — adds the `--dolphin`/`-D` flag (table entry, short-opt
  string, parsing case, cleanup)
- `src/platform/sdl/main.c` — adds a `connectToDolphin()` helper that calls the existing
  `GBASIODolphinCreate` → `GBASIOSetDriver` → `GBASIODolphinConnect` sequence right after
  a ROM successfully loads

## Building for H700 devices

### 1. Get a matching cross-toolchain

Grab an SDK tarball matching your firmware's release from your CFW's toolchain
releases (e.g. `knulli-cfw/toolchains` on GitHub). **The toolchain generation should
match your flashed firmware's generation** — an old toolchain against a newer rootfs
(or vice versa) can cause GLIBC symbol-version mismatches at runtime.

If no prebuilt SDK matches your firmware version, you can build one from source using
your CFW's Docker-based build system (most Batocera-derived projects, KNULLI included,
document this on their wiki) — clone the firmware's source repo, check out the tag
matching your installed version, and generate the SDK from that build.

```bash
tar xf your-toolchain-sdk.tar.gz
cd <extracted-folder>
./relocate-sdk.sh
export PATH=/path/to/toolchain/bin:$PATH
export CC=aarch64-buildroot-linux-gnu-gcc      # confirm exact prefix for your SDK
export CXX=aarch64-buildroot-linux-gnu-g++
```

Sanity-check before going further:

```bash
echo 'int main(){return 0;}' > hello.c
$CC hello.c -o hello
file hello   # should report ARM aarch64, not your host's architecture
```

Copy `hello` to the device and confirm it runs over SSH before touching mGBA — this
isolates toolchain problems from build problems.

### 2. Write a CMake toolchain file

```cmake
# h700-toolchain.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(TOOLCHAIN_ROOT /path/to/extracted/sdk)
set(TOOLCHAIN_PREFIX aarch64-buildroot-linux-gnu)

set(CMAKE_C_COMPILER   ${TOOLCHAIN_ROOT}/bin/${TOOLCHAIN_PREFIX}-gcc)
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_ROOT}/bin/${TOOLCHAIN_PREFIX}-g++)

set(CMAKE_FIND_ROOT_PATH ${TOOLCHAIN_ROOT}/${TOOLCHAIN_PREFIX}/sysroot)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

set(ENV{PKG_CONFIG_PATH} "")
set(ENV{PKG_CONFIG_LIBDIR} ${CMAKE_FIND_ROOT_PATH}/usr/lib/pkgconfig)
set(ENV{PKG_CONFIG_SYSROOT_DIR} ${CMAKE_FIND_ROOT_PATH})
```

### 3. Configure and build

```bash
mkdir build-h700 && cd build-h700
cmake -S /path/to/this/repo -B . \
      -DCMAKE_TOOLCHAIN_FILE=/path/to/h700-toolchain.cmake \
      -DBUILD_QT=OFF -DBUILD_SDL=ON \
      -DBUILD_GL=OFF -DBUILD_GLES2=OFF -DBUILD_GLES3=OFF -DUSE_EPOXY=OFF \
      -DUSE_FFMPEG=OFF -DUSE_LIBZIP=OFF -DUSE_LUA=OFF \
      -DUSE_JSON_C=OFF -DUSE_ELF=OFF -DUSE_SQLITE3=OFF -DUSE_DEBUGGERS=OFF
make -j$(nproc)
```

Notes on the flags:
- `BUILD_GL`/`BUILD_GLES2`/`BUILD_GLES3`/`USE_EPOXY` are all disabled to force the
  software renderer — the H700's Mali GPU doesn't support desktop GL, and a partial
  GL detection during cross-compile can leave a dangling `mSDLGLCreate` reference at
  link time. Software rendering is plenty for this use case.
- The `USE_*`/`BUILD_*` feature flags trim ffmpeg, libzip, Lua, ELF debugging, and
  sqlite3 — none of these are present on a typical CFW rootfs, and linking against
  them (even if your SDK's sysroot happens to have the `.pc`/headers) just produces
  a binary with dependencies that don't exist on the target device.

You'll get two files you need: `sdl/mgba` (the executable) and `libmgba.so.0.11`
(built into the top of the build directory). **Both must be copied to the device
together** — `libmgba.so` contains all the core logic including the CLI parsing
changes, so any edit to `commandline.c` requires re-copying it, not just the
`mgba` binary.

## Installing as a KNULLI/Batocera "port"

```bash
mkdir -p /userdata/roms/ports/mgba-dolphin
# copy mgba, libmgba.so.0.11(*), your BIOS file, and your ROM(s) here
```

Create `/userdata/roms/ports/MGBA-Dolphin.sh`:

```bash
#!/bin/bash
DIR="/userdata/roms/ports/mgba-dolphin"
export LD_LIBRARY_PATH="$DIR:$LD_LIBRARY_PATH"
cd "$DIR"
./mgba --bios "$DIR/gba_bios.bin" --dolphin <YOUR-PC-IP> "$DIR/yourgame.gba"
```

```bash
chmod +x /userdata/roms/ports/MGBA-Dolphin.sh
```

Rescan/restart ES and launch it from the **Ports** section of the menu — launching
this way (rather than manually over SSH) is what gives you correct display and
controller-input handoff. Manually killing EmulationStation and running the binary
over SSH will leave you with no working controller input.

## Usage notes (read this before assuming something's broken)

- **A GBA BIOS file is required.** Without `--bios`, the connection can appear to
  succeed at the socket level but the in-game link handshake will never actually
  complete. This isn't unique to this patch — it trips up the official desktop
  Qt build too.
- **The GBA↔Dolphin link is genuinely unreliable, even in the official upstream
  build.** mGBA's own 0.9.0 release notes describe it as "generally unreliable,"
  and there are long-standing upstream GitHub issues (e.g. #2210, #3065) about
  silent connection hangs and network-related instability. If it doesn't connect
  on the first try, that matches expected behavior, not a bug in this patch.
- **Try soft-resetting the GBA (A+B+Start+Select) after connecting**, if the
  in-game screen is stuck waiting. Some titles (Pokémon Box, Colosseum, XD) appear
  to need the GBA to reboot *after* the TCP link is already established for the
  handshake to latch — this is the manual SDL-frontend equivalent of the Qt
  frontend's "reset mGBA when it connects" checkbox.
- **Set Dolphin's relevant GameCube controller port to `GBA (TCP)`** *before*
  loading the game, and make sure Dolphin/the game are freshly restarted rather
  than reused from a previous failed attempt — stale state on either side seems
  to make the handshake less likely to succeed.
- Tested with **Pokémon Box** (no game-progress requirement, good for a first
  connectivity test) and should work with any GBA↔GameCube-linkable title —
  Pokémon Colosseum/XD, Four Swords Adventures, Crystal Chronicles, etc. — subject
  to the same network-latency sensitivity in the underlying protocol.

## Known limitations

- Only tested on RG34XX / KNULLI. Should be portable to other H700 CFWs (muOS,
  ArkOS, JELOS) with a matching toolchain, but paths/packaging conventions differ.
- No in-app UI for entering the Dolphin IP — it's a fixed CLI argument, set at
  launch. A future improvement could read it from a config file instead.
- The underlying link protocol (inherited from VBA) is not designed for real
  network latency, only same-machine use — see upstream discussion in
  [mgba-emu/mgba#3065](https://github.com/mgba-emu/mgba/issues/3065). Over WiFi,
  expect occasional connection flakiness; this matches official-build behavior,
  not a regression introduced here.

## Credits

Built on [mGBA](https://github.com/mgba-emu/mgba) by endrift and contributors.
This fork only adds a thin CLI wrapper around functionality that already existed
in mGBA's core and Qt frontend.
