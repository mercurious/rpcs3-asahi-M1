# rpcs3-asahi-M1

**RPCS3 (ETK GTK Edition) built natively for Asahi Linux on Apple M1, running on the
open-source Honeykrisp Vulkan driver** — a second independent open-driver validation surface
for the ETK GT-tuning work, alongside the Adreno 650 / Mesa Turnip handheld rig.

This is **not a fork of RPCS3** and **not a shipping / performance target**. It is a build
recipe + validation record: upstream RPCS3 + the
[etk-rpcs3-gtk](https://github.com/mercurious/etk-rpcs3-gtk) GTK-Edition patch + one
Asahi-surfaced fix, plus the runtime tuning that makes it render on an 8 GB unified-memory
laptop. Everything needed to reproduce the exact binary is in [`BUILDING.md`](BUILDING.md);
provenance and sha256 in [`PROVENANCE.md`](PROVENANCE.md).

## Why an M1 surface

Mesa **Turnip** is Adreno reverse-engineered; Asahi's **Honeykrisp** is Apple's M1 GPU
reverse-engineered (Alyssa Rosenzweig / Asahi Lina). Two GPUs, two clean-room open drivers,
one emulator patch set. A fork behaviour that reproduces on **both** is a property of RPCS3 +
the guest, not one driver's quirk; a behaviour on only one is a driver-frontier signal. This
surface earned its keep on day one (below).

## What this surface found (2026-08-31 standup)

| finding | result |
|---|---|
| **clang-22 SPU `stop_and_signal` miscompile** | GT5P Spec II (NPUA80075) segfaults on stock at the documented garbage pointer `0x00191a1b1c1d1e27` — **reproduced on Honeykrisp/aarch64**, confirming it is compiler-driven, not x86-specific. The `optnone` fix in the 0.8.4 patch resolves it and GT5P boots. |
| **Road-flicker fix (`GTK_REMAP0`)** | fires on Honeykrisp with the same `TIU15 raw=0x00000000` signature as Adreno — the fix cross-validated on another open Vulkan. |
| **Fence-force anti-lock net** | correctly detects a real Honeykrisp GPU stall and surfaces the player rescue overlay. |
| **Double rescue overlay** | genuine fork bug found + fixed here (`patches/overlay-coalesce-notice.patch`): the notice was queued from two fence paths per stall with no coalescing. |
| **KDE-Wayland explicit-sync crash** | stock RPCS3 crashes creating its render window on Plasma-Wayland + Honeykrisp (`wp_linux_drm_syncobj`); XWayland workaround. |
| **Sparse-reservation OOM** | Fedora's default overcommit refuses RPCS3's ~32 GiB VM reservation on 8 GB; `vm.overcommit_memory=1` fixes it. |
| **Unified-memory VRAM budget** | RPCS3's 64 GB default `VRAM allocation limit` is a footgun on unified memory; cap it (see BUILDING §5.3). |
| **Controller hidraw gap** | Asahi ships no controller `uaccess` rules, so the DualSense's `/dev/hidraw*` is root-only — breaking *both* RPCS3 pad handlers (native reads hidraw; SDL3 HIDAPI claims-then-fails, never falling back to the accessible evdev node). Fixed by `tools/70-dualsense.rules`; DualSense then works. |
| **Resolution-Scale pink screen** | RPCS3's internal Resolution-Scale (%) upscale renders magenta on Honeykrisp — keep Scale at 100 % and lower the *base* resolution instead. |

## Status

- **AfterBurner Climax** — playable, near console quality once the desktop is lean; DualSense working (SDL handler + `tools/70-dualsense.rules`).
- **GT5P** — boots to menu, dealership, and the car-purchase flow on Honeykrisp; heaviest transients hit the 8 GB-unified + desktop memory ceiling (`overcommit=1` + VRAM cap + 480p base res reach furthest; Resolution-Scale % pinks — use base res).
- **GTHD** — boots; its heavy render configs vs. the unified-memory budget are still being balanced.

A bench oracle, off the release path — for revalidating fork logic on a totally different GPU.
Not part of the ETK release stack or the certified-cores catalog.

## Quickstart

```sh
# toolchain + native build: see BUILDING.md
sudo cp tools/99-rpcs3-overcommit.conf /etc/sysctl.d/ && sudo sysctl --system
QT_QPA_PLATFORM=xcb ./build/bin/rpcs3
# or install the KDE launcher (bakes in xcb):  tools/rpcs3-gtk.desktop
# lean track-test mode (quit desktop co-tenants): tools/etk-track-mode
```

## License

GPL-2.0 — this distributes a build of RPCS3. See [`LICENSE`](LICENSE) and [`PROVENANCE.md`](PROVENANCE.md).
