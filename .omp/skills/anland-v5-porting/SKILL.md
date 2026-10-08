---
name: anland-v5-porting
description: Port Linux WMs/compositors to the Mango native Anland 5 stack in this repository. Use for Anland backend design, build order, packaging boundaries, runtime startup, and verification against the pinned wlroots/SceneFX/Anland/Mango anland5 branches.
---

# Anland v5 WM/compositor porting

Use this skill when adding, reviewing, or packaging a Linux WM/compositor backend for the Mango native Anland 5 stack.

## Ground truth for this repository

This project is a downstream integration snapshot, not an upstream replacement for Mango or Anland. Treat product code in the submodules as the source of truth; the top-level repository pins a known stack and documents how to rebuild it.

Pinned stack from the root README:

| Order | Submodule | Branch | Pinned commit | Package/output role |
|---:|---|---|---|---|
| 1 | `wlroots/` | `anland5` | `f2ff3e6c040f474ca71e8a5639997934f7a99142` | wlroots 0.20.2 plus external swapchain API, packaged as `libwlroots-0.20` |
| 2 | `scenefx/` | `anland5` | `612c23fa80106ae0968a417ac711f2e4e8b4c9ef` | SceneFX 0.5 built against the pinned wlroots, packaged as `libscenefx-0.5-0` |
| 3 | `anland/` | `anland5` | `b5ffb976c2e263701be6286a4afeadef3633d92a` | `display-producer` 5.0.0, Debian revision `5.0.0-2anland5`, runtime `libdisplay-producer5` |
| 4 | `mango/` | `anland5` | `a6b0606a7db5ebd15426aa7e95c6155006f7e675` | Mango 0.17.5, Debian revision `0.17.5-2anland5`, runtime `mango-anland5` |

Current reported platform: Lenovo Xiaoxin Pad Pro GT, Debian 13, matching Mesa/Freedreno stack. Reported features: visual output, touch, mouse, touchpad, speakers, microphone, keyboard, bidirectional clipboard. Do not generalize this report to untested hardware, Mesa stacks, WM versions, or package builds.

## Architecture boundaries

1. Keep one Anland backend per WM/compositor. Put distribution differences in version patches, dependency locks, packaging recipes, and startup configuration.
2. Keep transport, protocol framing, session lifecycle, ACK handling, buffer release, backpressure, and daemon/consumer recovery in the common Anland producer layer.
3. The WM still composites the complete desktop. Do not convert application windows into Android surfaces.
4. Do not modify the Android consumer or daemon protocol unless the task explicitly targets that component.
5. Build, packaging, installation, and runtime smoke are separate steps. Never let a build script kill/restart the user’s current WM, PipeWire, daemon, or consumer.
6. Preserve the repository’s four-component package model unless explicitly asked to design a different distribution artifact.

Data path from the README’s base protocol:

```text
WM native renderer, frame clock, output, seat, clipboard/text APIs
  -> thin Anland backend adapter
  -> libdisplay_producer / display-producer public APIs verified in headers
  -> existing daemon socket and deposited fds
  -> existing Android consumer-owned dmabufs, shm selected-index page, eventfd, fence socketpair, and data channel
```

This pinned revision exports the shared `anland_de_backend_commit/present/pump/dispatch` facade and `anland_de_backend_get_writable_target()` through `include/anland/anland_de_backend.h`. Use that boundary in WM adapters; do not duplicate raw ACK/shared-page/transport state in Mango. Verify contracts against headers when porting another revision.

## Before changing code

Create a short porting table before implementation:

- Target WM/compositor version or full commit.
- Anland producer revision and required public APIs.
- Renderer and external DMA-BUF import path.
- Output/mode/frame-clock entry points.
- Event loop fd watcher and retry timer APIs.
- Keyboard, pointer, touch, clipboard, and text-input injection APIs.
- Required desktop components: shell, portals, Xwayland, audio, session manager.
- Target distribution, architecture, toolchain, GPU driver, and render node.

Check the real public headers before using a suggested API. `display-producer` is versioned as 5.0.0 in the README, but different anland5 revisions may not expose newer helper APIs.

Anland `README.md` for this branch adds hard interface facts that must be preserved in porting work: `libdisplay_producer` installs with SONAME 5; public producer headers install under `${prefix}/include/anland`; the pkg-config module name is exactly `display-producer`; expected module version for this snapshot is `5.0.0`.

V3 clipboard support is API/wire incompatible with V2 even when fixed-size structs keep the same size. `INPUT_TYPE_CLIPBOARD` and `OUTPUT_TYPE_CLIPBOARD` carry a fixed event header followed by `clipboard.size` payload bytes. Receivers must either consume the payload with the matching `*_extend_data()` API or call `handle_unhandled_event()` for unprocessed variable-length events; leaving trailing bytes corrupts the data channel.

## Build and package order

Follow the root README order. Keep all components on one prefix/sysroot:

1. Build/install or package `wlroots/` first.
2. Build/install or package `scenefx/` against that wlroots.
3. Build/install or package `anland/` and confirm `pkg-config --modversion display-producer` returns `5.0.0` for this snapshot.
4. Build/install or package the WM/compositor, e.g. Mango with `-Danland=enabled` and `-Dxwayland=enabled`.

Manual install may use `/usr/local`; Debian packaging uses `/usr` and the host multiarch libdir. Do not mix `/usr`, `/usr/local`, staged installs, and old build artifacts in one verification claim.

For Debian packaging in this repository:

| Submodule | Source package | Runtime package | Development package |
|---|---|---|---|
| `wlroots` | `wlroots` | `libwlroots-0.20` | `libwlroots-0.20-dev` |
| `scenefx` | `scenefx` | `libscenefx-0.5-0` | `libscenefx-0.5-dev` |
| `anland` | `anland` | `libdisplay-producer5` | `libdisplay-producer-dev` |
| `mango` | `mango` | `mango-anland5` | — |

`mango-anland5` build-depends on the pinned wlroots, SceneFX, and Anland development packages. If a target WM is not available in the distribution, package it natively for that distribution; do not default to `sudo make install` or untracked files under `/usr`.

Debian 13 stable does not satisfy all current Build-Depends. Prefer compatible backports for wayland-protocols >= 1.47 and xkbcommon >= 1.8; obtain or backport Debian packages for Wayland >= 1.24, libdrm >= 2.4.129, and pixman >= 0.46 if the configured repositories are too old. Run `dpkg-checkbuilddeps` before each component build; a private-prefix install does not satisfy dpkg package dependencies. Build native `.deb` artifacts with `dpkg-buildpackage -b -us -uc`, installing prerequisite runtime/dev packages only in the prepared build environment.

## Backend implementation rules

### Output and buffers

- Treat consumer buffers as externally owned render targets.
- Track imports by session generation and slot index.
- Validate target index/count, width, height, stride, offset, format, modifier, and overflow.
- Render the full desktop into the actual writable target selected for that frame.
- Re-check session/generation/target after rendering and before commit.
- Commit only the buffer that was rendered.
- First version should prefer full-frame repaint over fragile buffer-age/damage optimization.

Frame sequence from `anland/README.md`:

```text
consumer selects dmabuf index in shm and signals buf_ready_efd
  -> producer reads selected index
  -> producer renders complete desktop into dmabuf[index]
  -> producer exports/stashes render fence with set_render_fence(fence_fd), or explicitly synchronizes
  -> producer calls trigger_refresh()
  -> consumer refresh_done() receives completion byte and optional fence fd
```

Do not mark a native frame presented because the daemon handshake succeeded, because the data socket is writable, or because the buffer-ready fd woke up. The README’s base protocol exposes selected-buffer notification and render completion; any higher-level commit/present/drop/release abstraction must be verified in the current public headers before use.

### Event handling

- Use the producer library’s documented input/data APIs for input and clipboard; do not invent private data-channel framing in a WM backend.
- Query `anland_de_backend_get_writable_target()` before acquiring, scheduling or committing a buffer. It rejects stale generations, mismatched raw slots, retained slots, accepted/in-flight commits and release-publication retries.
- Keep active commit state separate from per-slot submitted buffer locks. PRESENTED ends the commit; only the matching BUFFER_RELEASED permits slot reuse.
- Pump then drain outcomes/releases even when pumping reports an error. Retry publication backpressure while still connected; clean event-owned fence fds in the entire returned batch before teardown on error.
- Transport pre-release callbacks must not re-enter the producer or WM. Detach callbacks/watchers there; the owning event loop drains outcomes, resets input, and retires imports.

### Sync and fd ownership

- Borrowed fds from device/watchers must not be closed by the WM.
- Buffer description fds returned to the caller are caller-owned.
- `set_render_fence()` takes ownership of its fd in this revision. Do not close it after handing it off; check the exact API's transfer/duplication contract before using other fence APIs.
- Fences returned by consumer-side `refresh_done()` belong to that receiver; import/wait as required, then close exactly once.
- `-1` means no explicit fence, not “ignore unfinished GPU work”.
- Release fence fds returned by `anland_de_backend_dispatch()` are caller-owned. Import into the matching live buffer before reuse, then close; also close ignored/remaining batch fds on failure.
- Only explicit unsupported ioctl errors may fall back to implicit sync. Preserve errno before cleanup; EINVAL is not proof of unsupported sync.

### Input, clipboard, and text

- Convert Anland keyboard, pointer, axis, and touch events through the WM’s native seat/input APIs.
- On disconnect/destruction, release pressed keys/buttons and cancel live touch points.
- Bridge clipboard through the WM’s selection/MIME APIs. Avoid bidirectional loops.
- Variable-length text/clipboard payloads require bounded lengths, nonblocking/progress-aware reads, and explicit recovery for short reads, invalid fds, queue full, and disconnect.
- Prefer the WM’s text-input/IME path for committed Unicode text. Do not turn arbitrary Unicode into global keymap changes unless the WM has a client-scoped, reviewed fallback mechanism.
- Direct Android text-input-v3 is enabled only for the actual Anland backend. Require enabled input, matching relay/text-input/seat focus, correct client ownership, valid UTF-8 and no IME keyboard grab. On IME startup/restart, activate once and replay the current editor state before done.

## Runtime startup

The README-defined Mango session selects the Anland backend with environment variables:

```sh
export ANLAND_SOCKET=/run/display.sock
export ANLAND_DRM_DEVICE=/dev/dri/renderD128
export XDG_RUNTIME_DIR=/run/user/$(id -u)
exec mango -s 'mangobar'
```

Installed Mango package flow:

```sh
systemctl --user enable --now mango-anland.service
mango-anland {start|stop|restart|kill|status}
anland-volume.sh {get|up|down|toggle|set <0-150>}
```

For other WMs, create a separate startup entry that validates runtime directory, daemon socket, render device, library paths, and required desktop services. Startup scripts should refuse unsafe takeover of an existing session unless explicitly authorized.

## Verification standard

Static success is not runtime success. Use only observed evidence.

Minimum non-graphical checks from the README pattern:

```sh
(cd wlroots && git diff --check)
(cd scenefx && git diff --check)
(cd anland && git diff --check)
(cd mango && git diff --check)
sh -n mango/scripts/mango-anland mango/scripts/anland-volume.sh
```

Build evidence must name the actual build tree, prefix/sysroot, package versions, dynamic links/RPATH, and input revisions. A binary from a stale build tree is not evidence for current source.

Runtime smoke needs explicit graphical authorization and should cover:

- first frame and sustained redraw,
- window movement/effects/cursor correctness,
- pointer, keyboard, touch, clipboard, and text input,
- consumer reconnect and daemon recovery,
- resize/rotation or output mode change,
- required Wayland and Xwayland clients,
- no obvious fd growth, duplicate release, stuck pressed state, or stale generation reuse.

Do not claim the root README’s reported Lenovo test as proof for a new backend, new package, new device, or newly edited code. Report unsupported or untested items explicitly.

## Packaging and release notes checklist

A complete port deliverable should include:

- one WM backend implementation,
- minimal version-specific compatibility patches,
- source/dependency lock data,
- feature matrix with tested/untested status,
- reproducible build recipe,
- native package recipe,
- isolated startup entry,
- verification log with exact commands and observed outputs.

If the task is only assessment or review, do not edit product code. Provide exact paths/symbols, observed behavior, recommended change, and an observable acceptance scenario.