# rpcs3-asahi-M1 — provenance

This repo is **not** a fork of RPCS3. It is a downstream *build recipe + validation record* for
running the ETK RPCS3 GTK Edition natively on **Asahi Linux / Apple M1** against the open-source
**Honeykrisp** Vulkan driver — a second open-driver surface alongside the Adreno 650 / Mesa
Turnip rig. The emulator source is upstream RPCS3; the delta is the GTK-Edition patch (from
etk-rpcs3-gtk) plus one Asahi-surfaced fix.

## Corresponding source (GPL-2.0)

| layer | value |
|---|---|
| upstream base | RPCS3 `c6e96729c` — `v0.0.42`, build **19895** |
| GTK patch | `patches/etk-rpcs3-gtk-edition-0.8.4-dev.patch` (from [etk-rpcs3-gtk](https://github.com/mercurious/etk-rpcs3-gtk)) |
| Asahi fix | `patches/overlay-coalesce-notice.patch` |
| build recipe | [`BUILDING.md`](BUILDING.md) |
| prerelease binary | `rpcs3-etk_gtk-0.8.4-overlayfix_v0.0.42-19895_asahi_aarch64` |
| sha256 | `9527cf0687afb063b5e435da07bca094fef0cf078517284259602014cf1cb8b9` |

Apply the two patches in order (`0.8.4-dev`, then `overlay-coalesce-notice`) onto `c6e96729c`.

## Notes

- The GTK `0.8.4-dev` patch was authored against upstream `a1deb2921` (`v0.0.41-19638`) for the
  rig, but **applies cleanly to `19895`** (0 rejects) — verified in this build.
- `overlay-coalesce-notice.patch` is an **Asahi-surfaced fix**: the "GTK Crash Recovery" rescue
  overlay was queued from two fence paths per stall (the warn in `etk_fence_note_not_ready` and
  the resolution in `etk_fence_note_signaled`) with no coalescing, stacking duplicate overlays.
  First observed on Honeykrisp, 2026-08-31. A candidate for folding into the etk-rpcs3-gtk
  cumulative patch.
- The binary self-reports the stock upstream version string (`0.0.42-19895`) — the patch set
  does not bump it. Identify builds by **sha256** and the `GTK Edition v0.8.4-dev` banner in the
  log, not the About dialog.
- Marker verification: `etk-rpcs3-gtk/scripts/verify-markers.sh <patch> <binary>` reports all 13
  fork markers present in this build (patch-level and binary-level).
