# StreamX-LibVLC

**A deep-patched fork of libVLC, purpose-built as the secondary playback engine for [StreamX Ultra](https://github.com/AeonCoreX-Lab).**

This is not a vanilla libVLC redistribution. This fork strips, rewires, and extends upstream VLC media framework to serve one specific purpose: **consuming live, partially-downloaded torrent data as a native VLC input source**, with Android as the sole target platform.

> Part of the AeonCoreX-Lab ecosystem. Sibling project to `torrent-engine` (libtorrent/JNI) and `MpvHandler` (primary playback engine).

---

## Why This Fork Exists

StreamX Ultra already uses **MPV** (via `libmpv` + custom JNI `MpvHandler.cpp`) as its primary player. That engine is fast, GPU-render-controllable, and well-integrated with our torrent piece-buffer.

MPV is not the right tool for every situation, though. Some containers/codecs/network conditions are handled more gracefully by VLC's mature stream-resilience logic and broader demuxer coverage. Rather than compromise MPV's lean footprint by over-engineering it to handle every edge case, StreamX Ultra runs **dual playback engines**, falling back to VLC when MPV can't cleanly handle a stream.

Stock libVLC / libvlc-android is not designed to accept a live, growing, out-of-order byte source (a torrent download in progress). It expects files, local sockets, or standard network protocols. **This fork's core job is to make VLC natively understand our torrent piece buffer as a first-class input**, the same way it already understands HTTP or a local file.

---

## Relationship to StreamX Ultra Architecture

```
                    ┌──────────────────────────┐
                    │  PlayerManager.kt          │
                    │  (engine selection/fallback)│
                    └─────────────┬─────────────┘
                                  │
              ┌───────────────────┴───────────────────┐
              │                                         │
   ┌──────────▼───────────┐                ┌───────────▼────────────┐
   │   MpvHandler.cpp        │                │   VlcHandler.cpp          │
   │   (existing, JNI)         │                │   (this fork's JNI bridge)  │
   └──────────┬───────────┘                └───────────┬────────────┘
              │                                         │
   ┌──────────▼───────────┐                ┌───────────▼────────────┐
   │   libmpv (stock)        │                │   libvlc (THIS FORK)       │
   └──────────┬───────────┘                └───────────┬────────────┘
              │                                         │
              │                    ┌────────────────────▼───────────────┐
              │                    │  access_torrentbuffer.c (NEW MODULE)  │
              │                    │  Custom VLC access module              │
              │                    └────────────────────┬───────────────┘
              │                                         │
              └───────────────────┬─────────────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │   TorrentEngine.cpp (C++)     │
                    │   libtorrent session, shared    │
                    └────────────────────────────┘
```

Both engines read from the **same underlying torrent piece buffer**. Neither engine owns or duplicates download logic — `TorrentEngine.cpp` remains the single source of truth for piece availability, sequential download prioritization, and buffer state.

---

## Scope of the Deep Patch

This is tracked work, not a one-shot rewrite. Patching happens in four layers, in order:

### Layer 1 — Build Surface Reduction
Stock `libvlc-android` bundles far more than StreamX Ultra needs (DVD/CD/Blu-ray input, radio tuners, a long tail of rarely-used audio codecs, VLC's own HTTP/RTSP servers, etc). Before any feature patching, the build is trimmed to a video-streaming-only footprint.

- Target: **30–40% APK size reduction** vs stock `libvlc-android` AAR
- Method: `modules/access/`, `modules/services_discovery/`, and `modules/stream_out/` pruned via `configure` flags and `Android.mk` / meson module exclusion lists
- Removed wholesale: DVD/VCD/CDDA access, Chromecast sout, VNC, satellite/DVB, most legacy subtitle demuxers we don't encounter in torrent releases

### Layer 2 — Custom Access Module: `access_torrentbuffer`
The centerpiece of this fork. A new VLC **access module** (`modules/access/torrentbuffer.c`) that VLC's input thread can open via a custom MRL scheme (`torrentbuf://`), backed directly by `TorrentEngine.cpp`'s piece buffer rather than a file descriptor or socket.

Responsibilities:
- Implements VLC's `stream_t` read/seek callbacks against the live piece buffer
- Respects piece-availability state — blocks/stalls gracefully (not a hard error) when a seek lands on a not-yet-downloaded region, and signals `TorrentEngine` to reprioritize that piece
- Reports buffering state upward through VLC's stats API so `VlcHandler.cpp` can surface real buffering UI, not just a spinner
- Thread-safety boundary against the JNI layer — this is the layer most prone to subtle bugs, so it gets the most test coverage

### Layer 3 — JNI Bridge: `VlcHandler.cpp`
Mirrors the existing `MpvHandler.cpp` command surface so `PlayerManager.kt` can treat both engines nearly interchangeably:
- `play() / pause() / seekTo() / setSubtitleTrack() / setAudioTrack() / getBufferedRanges()`
- Exposes VLC's `libvlc_media_player_t` event callbacks (buffering, error, end-reached) across JNI to Kotlin

### Layer 4 — Fallback & Selection Logic
Lives in `PlayerManager.kt` (StreamX Ultra repo, not this repo) — decides per-stream which engine to launch, and handles live failover if MPV throws a demux/codec error mid-playback.

---

## What This Fork Does *Not* Do

Being explicit about boundaries, since scope creep is the main risk on a project like this:

- **No UI code.** All playback surfaces (controls, subtitle rendering UI, gesture handling) stay in the StreamX Ultra Kotlin/Compose layer, same as with MPV.
- **No changes to libtorrent or `TorrentEngine.cpp`.** This fork *consumes* the existing piece buffer interface; it doesn't modify torrent logic. Any interface changes needed on that side are tracked as issues against `torrent-engine`, not patched here.
- **No general-purpose VLC distribution.** This is not meant to be a drop-in replacement for `libvlc-android` in other projects. Compatibility with standard libVLC input types (plain HTTP, local files) is preserved where free, but not prioritized when it conflicts with the torrent-input goal.
- **No Windows/Linux/desktop targets.** Android only, matching StreamX Ultra's actual deployment surface.

---

## Repository Layout (planned)

```
streamx-libvlc/
├── vlc/                        # upstream libVLC source (git subtree/submodule, pinned commit)
├── patches/                    # sequential .patch files applied over upstream, in order
│   ├── 0001-strip-unused-access-modules.patch
│   ├── 0002-add-torrentbuffer-access-module.patch
│   ├── 0003-buffering-stats-passthrough.patch
│   └── ...
├── modules/access/
│   └── torrentbuffer.c         # the new access module (lives here pre-merge, applied via patch)
├── jni/
│   └── VlcHandler.cpp          # JNI bridge, mirrors MpvHandler.cpp conventions
│   └── VlcHandler.h
├── build/
│   ├── android-toolchain.cmake
│   └── module-exclusions.txt   # Layer 1 trim list
├── .github/workflows/
│   └── android-build.yml       # CI: applies patches, cross-compiles, produces .aar
├── docs/
│   ├── ARCHITECTURE.md
│   └── ACCESS_MODULE_PROTOCOL.md   # torrentbuf:// MRL spec, buffer handoff contract
└── README.md
```

Patches are kept as a **sequential series against a pinned upstream commit**, not a hard fork with divergent history. This keeps rebasing onto newer upstream VLC security fixes tractable — a real concern, since VLC ships CVE fixes regularly and we don't want to silently fall behind on media-parsing security patches given this handles untrusted network/torrent input.

---

## Build & CI

GitHub Actions handles cross-compilation to Android (`arm64-v8a` primary target; `armeabi-v7a` as a secondary target for older devices). The workflow:

1. Checks out pinned upstream VLC commit
2. Applies `patches/*.patch` in sequence
3. Runs Layer 1 module-exclusion build config
4. Cross-compiles via Android NDK toolchain (matching the NDK version StreamX Ultra's `torrent-engine` already targets, for ABI consistency)
5. Packages output as `.aar` / `.so` artifacts
6. Runs the module protocol smoke test (`docs/ACCESS_MODULE_PROTOCOL.md` conformance check) against a synthetic partial-buffer fixture

Build is **not** a one-shot script — each patch layer is independently buildable and testable, so a regression in Layer 2 doesn't block validating Layer 1's size reduction.

---

## Official Upstream

This fork is based on the official VLC media framework source:

- **Primary (GitLab)**: https://code.videolan.org/videolan/vlc
- **Android wrapper (base for this fork)**: https://code.videolan.org/videolan/vlc-android
- **GitHub mirror (read-only)**: https://github.com/videolan/vlc

---

## Credits & Attribution

This project is a derivative work. None of the core media playback engineering here is original — full credit belongs upstream:

- **VLC media player / libVLC** — © [VideoLAN](https://www.videolan.org/) and the VLC authors/contributors. VLC is a registered trademark of the VideoLAN non-profit organization. This fork is an independent derivative and is **not affiliated with, endorsed by, or supported by VideoLAN**. See `vlc/AUTHORS.txt` and `vlc/COPYING.txt` (preserved unmodified from upstream) for the full contributor list and license text.
- **vlc-android** — the Android build tooling, NDK toolchain integration, and Gradle/JNI scaffolding this fork's Android build process is based on comes from the official `vlc-android` project, also © VideoLAN and contributors.
- Any third-party libraries bundled by upstream VLC (codec libraries, demuxers, etc. under `vlc/contrib/` or `thirdparty/`) retain their own original licenses — see upstream's `COPYING.LGPL`, `COPYING.LIB`, and per-library notices. This fork does not alter those licenses.

If this project is published or distributed, the upstream `LICENSE.txt` / `COPYING.txt`, `AUTHORS.md`, and `COPYRIGHT.txt` files **must remain intact and unmodified** in the `vlc/` subtree, per LGPL §4/§6 obligations.

---

## License

libVLC is licensed **LGPL v2.1**. This fork remains LGPL-compliant:
- All patches against upstream VLC source are published in `patches/`, satisfying LGPL's modification-disclosure requirement
- `VlcHandler.cpp` (the JNI bridge) links against libVLC as a shared library, not statically, preserving LGPL's dynamic-linking exemption for the proprietary parts of StreamX Ultra
- Upstream `COPYRIGHT.txt` and `LICENSE.txt` are preserved unmodified in `vlc/`

**Do not statically link this fork's output into a closed-source binary without legal review** — that's the one LGPL trap worth flagging explicitly here.

---

## Status

🚧 **Planning stage.** This README documents the target architecture before implementation begins. Current milestone: Layer 1 (build surface reduction) + upstream commit pinning.

Tracking issues will be filed per layer as work starts.
