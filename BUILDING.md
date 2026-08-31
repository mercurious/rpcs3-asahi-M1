# Building RPCS3 for Asahi Linux / Apple M1 (Honeykrisp)

A native aarch64 build of the ETK RPCS3 GTK Edition on **Asahi Linux (Fedora Remix)**, running
on the open-source **Honeykrisp** Vulkan driver. **This is a validation / bench target, not a
performance target** — an 8 GB unified-memory laptop is a different class from the handheld the
kit ships on.

Validated 2026-08-31 on a MacBook Air M1: Asahi Fedora 44 (`7.1.6-400.asahi.fc44.aarch64+16k`,
16 KB pages), Mesa 26.1.8 (Honeykrisp), Vulkan 1.4.341, clang/LLVM 22.1.8, Qt 6.11.1.

## 1. Toolchain (dnf)

```sh
sudo dnf install -y alsa-lib-devel cmake ninja-build glew-devel libatomic \
  libevdev-devel libudev-devel openal-soft-devel qt6-qtbase-devel \
  qt6-qtbase-private-devel qt6-qtmultimedia-devel qt6-qtsvg-devel \
  vulkan-headers vulkan-loader-devel libcurl-devel SDL3-devel \
  pipewire-jack-audio-connection-kit-devel llvm-devel clang ccache \
  libxkbcommon-x11-devel lld
```

Two packages beyond upstream's Fedora list, both surfaced on this build:

- **`libxkbcommon-x11-devel`** — RPCS3 auto-enables its X11 keyboard path when it detects
  X11/xcb, and that path needs `xkbcommon/xkbcommon-x11.h`. Without it the build fails at
  ~95 % on `Input/basic_keyboard_handler.cpp`.
- **`lld`** — RPCS3 defaults to `-flto=thin`; the emu-core objects are LLVM bitcode, which
  `ld.bfd` cannot read (`error adding symbols: file format not recognized` at the final link).
  `lld` reads bitcode natively.

Fedora's `llvm-devel`/`clang` are **22.1.8**. RPCS3 requires LLVM ≥ 18 (prefers 22.1) and
defaults `BUILD_LLVM=OFF`, so the **system LLVM is used and the multi-hour, memory-hungry
bundled-LLVM build is skipped entirely** — the single fact that makes this feasible on 8 GB.

## 2. Source + patches

```sh
git clone https://github.com/RPCS3/rpcs3 && cd rpcs3
git checkout c6e96729c            # v0.0.42, build 19895
# init submodules EXCEPT the bundled LLVM (system LLVM is used — saves a huge fetch):
git -c submodule."3rdparty/llvm/llvm".update=none submodule update --init --recursive
git apply /path/to/patches/etk-rpcs3-gtk-edition-0.8.4-dev.patch
git apply /path/to/patches/overlay-coalesce-notice.patch
```

## 3. Configure

```sh
CC=clang CXX=clang++ cmake -B build -G Ninja -DUSE_NATIVE_INSTRUCTIONS=OFF -DBUILD_LLVM=OFF
```

- **`-DUSE_NATIVE_INSTRUCTIONS=OFF`** is required on ARM (upstream `BUILDING.md`) or the SIMD
  headers `SIGILL`.
- Confirm the configure log shows `Using prebuilt or system LLVM` and `Found Vulkan … 1.4.341`.
  The `missing components: glslc glslangValidator` line is benign (RPCS3 builds vendored glslang).

## 4. Build (ThinLTO via lld, memory-capped)

```sh
nice -n 18 cmake --build build -j2
```

- **Cap `-j2`** on 8 GB — RPCS3 is one of the heaviest C++ builds and *memory*, not cores, is
  the wall (a desktop + browser will thrash it otherwise).
- With `lld` installed CMake links with it and ThinLTO works. If CMake did not pick up lld
  automatically, reconfigure with:
  `-DCMAKE_LINKER_TYPE=LLD -DCMAKE_EXE_LINKER_FLAGS="-Wl,--thinlto-jobs=2 -Wl,--thinlto-cache-dir=$PWD/build/thinlto-cache"`
  (the job cap keeps the ThinLTO link inside 8 GB; the cache makes re-links fast).
- Output: `build/bin/rpcs3` (~64 MB, unstripped). Full clean build ≈ 1.5–2.5 h at -j2;
  incremental (one file + ThinLTO relink) ≈ minutes.

## 5. Runtime — the Asahi-specific part (where "builds" becomes "renders")

Three settings turn *launches* into *runs games*. None are RPCS3 bugs; they are the honest cost
of a full desktop + a young open driver + unified memory.

### 5.1 Launch under XWayland — KDE-Wayland explicit-sync crash

On KDE Plasma Wayland + Honeykrisp, Qt 6.11's `wp_linux_drm_syncobj` (explicit sync) protocol
crashes when RPCS3 creates its game render window:
`wp_linux_drm_syncobj_surface_v1: error 1: Surface got destroyed already` → the Wayland
connection dies *before* PPU compilation. Run under XWayland to bypass the protocol:

```sh
QT_QPA_PLATFORM=xcb ./build/bin/rpcs3
```

(`tools/rpcs3-gtk.desktop` bakes this in. Native-Wayland alternative: `KWIN_DRM_NO_SYNCOBJ=1`
before KWin starts — compositor-wide, needs relogin.)

### 5.2 `vm.overcommit_memory = 1` — sparse-reservation OOM

RPCS3 reserves ~32 GiB of virtual address space (mostly never resident). Fedora's default
`vm.overcommit_memory=0` (heuristic) refuses that single reservation on an 8 GB box and the
OOM-killer takes rpcs3 despite a tiny real RSS. Set always-overcommit (the pages are lazy, so
only real resident use counts):

```sh
sudo cp tools/99-rpcs3-overcommit.conf /etc/sysctl.d/ && sudo sysctl --system
```

macOS (Mach VM) and the ROCKNIX rig do not restrict sparse VM this way, which is why only
Asahi Fedora hit this.

### 5.3 Cap the VRAM budget for unified memory

Apple Silicon has **unified memory** — Honeykrisp's GPU buffers come out of the same RAM as the
CPU side and are pinned (unswappable) DRM objects. RPCS3's default `VRAM allocation limit` of
**65536 MB** (64 GB — effectively "unlimited") lets it allocate GPU memory until it eats all
system RAM → `VK_ERROR_OUT_OF_DEVICE_MEMORY`, then a system OOM. Set the per-game (or global)
limit to leave headroom for the CPU side + desktop, e.g. **4096 MB**, tuning 3072–5120 by title
and by how lean the desktop is. On a discrete-VRAM GPU this default is harmless; on unified
memory it is a footgun. (RPCS3 does not yet budget against `VK_EXT_memory_budget`, which
Honeykrisp reports — a candidate upstream improvement.)

For a track test, quit the desktop co-tenants first: `tools/etk-track-mode` quits Firefox +
Claude Desktop and hands the terminal to the resumed Claude Code CLI session, so RPCS3 gets the
RAM the bare-metal rig gets for free.

### 5.4 Controllers — grant hidraw access (DualSense et al.)

Asahi Fedora KDE ships **no controller `uaccess` udev rules**, so a pad's raw HID node stays
`root:root 0600` — which breaks *both* of RPCS3's pad handlers: the native DualSense handler
opens `/dev/hidraw*` directly, and SDL3's HIDAPI driver *claims* the pad then fails to open
hidraw and never falls back to the (uaccess-tagged, accessible) `event*` node. Kernel + evdev
are fine; only hidraw is gated. Install the rule (DualSense `0ce6`, Edge `0df2`):

```sh
sudo cp tools/70-dualsense.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
# unplug/replug; verify:  getfacl -p /dev/hidrawN | grep dave   ->  user:dave:rw-
```

Then either handler works with full features. No-sudo stopgap: launch with
`SDL_JOYSTICK_HIDAPI=0` and the **SDL** handler (forces the evdev path — buttons/sticks, no
HIDAPI extras). Map **Options → Start** in the pad dialog (click the Start field, press Options).

## 6. Traps paid for — encode these, don't rediscover them

1. `libxkbcommon-x11-devel` missing → fails at ~95 % on `basic_keyboard_handler.cpp`.
2. ThinLTO bitcode + only `ld.bfd` → "file format not recognized" at the final link. Install `lld`.
3. KDE-Wayland explicit sync → render-window crash before PPU compile. `QT_QPA_PLATFORM=xcb`.
4. `overcommit_memory=0` → 32 GiB reservation refused → OOM at tiny RSS. Mode 1.
5. Unified VRAM + the 64 GB default limit → GPU allocations eat all RAM. Cap the VRAM budget.
6. Pad `/dev/hidraw*` is root-only (Asahi ships no controller `uaccess` rule) → both RPCS3
   handlers fail to see the controller. Install `tools/70-dualsense.rules` (§5.4).
7. Resolution-Scale (%) pinks the screen on Honeykrisp → keep it at 100 %, lower the base resolution.

The paths in `tools/` (`rpcs3-gtk.desktop`, `etk-track-mode`) are for the reference machine
(`/home/dave/rpcs3/build/bin/rpcs3`); adjust for your checkout.
