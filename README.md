# Mango Anland 5 stack

[中文](README_zh.md)

Run the Mango Wayland compositor through Anland on Android. This repository is a **downstream integration manifest**: four Git submodules pin the compositor and libraries that belong together. It is not a replacement for upstream Mango or Anland.

![Mango Anland 5 desktop and fastfetch](Screenshot_2026-10-05-22-44-32-28_048d46691f7e6d293bbd6f982d3d2551.jpg)

> **Experimental.** Much of the integration was written with AI assistance. Review the component diffs and rebuild locally. The reported working setup is a Lenovo Xiaoxin Pad Pro GT running Debian 13, with touch, mouse, touchpad, speakers, microphone, keyboard, and bidirectional clipboard. This is not a guarantee for other devices or builds.

## Contents

- [What you need to build](#what-you-need-to-build)
- [Clone the pinned stack](#clone-the-pinned-stack)
- [Install build dependencies](#install-build-dependencies)
- [Build Debian packages](#build-debian-packages)
- [Run a session](#run-a-session)
- [Verification and known limits](#verification-and-known-limits)
- [Repository policy and credits](#repository-policy-and-credits)

## What you need to build

**Build all four project components from their pinned sources. Install ordinary tools and system libraries through APT when their versions satisfy the package metadata.**

| Order | Component | Pinned commit | Debian runtime package | Debian development package | Pinned source version |
|---|---|---|---|---|---|
| 1 | [wlroots](wlroots/) | `f2ff3e6c040f474ca71e8a5639997934f7a99142` | `libwlroots-0.20` | `libwlroots-0.20-dev` | `0.20.2-1anland5` |
| 2 | [SceneFX](scenefx/) | `612c23fa80106ae0968a417ac711f2e4e8b4c9ef` | `libscenefx-0.5-0` | `libscenefx-0.5-dev` | `0.5.0-1anland5` |
| 3 | [Anland](anland/) | `b5ffb976c2e263701be6286a4afeadef3633d92a` | `libdisplay-producer5` | `libdisplay-producer-dev` | `5.0.0-2anland5` |
| 4 | [Mango](mango/) | `a6b0606a7db5ebd15426aa7e95c6155006f7e675` | `mango-anland5` | — | `0.17.5-2anland5` |

- **wlroots** contains the external swapchain API needed for consumer-owned Anland buffers. A stock distribution wlroots is not a substitute.
- **SceneFX** must be built against that wlroots ABI.
- **Anland** supplies the shared `display-producer` library and public headers.
- **Mango** must be built with its native Anland backend enabled.

SceneFX depends on wlroots. Mango depends on all three libraries. Anland can be built independently; the order above is a straightforward build sequence.

### Snapshot and ABI compatibility

The pinned Anland and Mango commits include the presentation/input lifecycle fixes and two permanent CPU regression tests. Anland adds writable-target APIs without changing SONAME 5; Mango requires the matching `5.0.0-2anland5` producer revision.

For any checkout, use its actual `debian/changelog` and `debian/control` as the authority for versions and exact dependencies. Do not mix a newer Mango package with the older producer package just because both expose pkg-config version `5.0.0`.

## Clone the pinned stack

```sh
git clone --recurse-submodules https://github.com/luochen88/mango-anland5.git
cd mango-anland5
```

For an existing clone without populated submodules:

```sh
git submodule update --init --recursive
```

Keep the recorded commits for a snapshot build. Updating to branch tips creates a different stack; it needs a fresh build and verification before its submodule pointers are published.

## Install build dependencies

Use a clean **Debian 13 build container or chroot**, preferably matching the target architecture. The reported target is `arm64`; these instructions assume native compilation, not cross-compilation. Keep the build environment separate from a running desktop.

### Tools and ordinary development packages

The current Debian recipes use Meson/Ninja for wlroots, SceneFX, and Mango, and CMake/Ninja for Anland. They enable GLES2 rendering, Xwayland, libliftoff, and XCB error support.

```sh
sudo apt update
sudo apt install \
  build-essential git dpkg-dev debhelper meson ninja-build cmake pkgconf \
  libwayland-dev libwayland-bin wayland-protocols \
  libdrm-dev libxkbcommon-dev libpixman-1-dev \
  libegl-dev libgbm-dev libgles-dev \
  libudev-dev libseat-dev libdisplay-info-dev libliftoff-dev \
  libinput-dev hwdata libpipewire-0.3-dev \
  libpcre2-dev libcjson-dev libpango1.0-dev \
  xwayland libxcb1-dev libxcb-dri3-dev libxcb-present-dev \
  libxcb-render0-dev libxcb-render-util0-dev libxcb-shm0-dev \
  libxcb-xfixes0-dev libxcb-xinput-dev libxcb-composite0-dev \
  libxcb-ewmh-dev libxcb-icccm4-dev libxcb-res0-dev \
  libxcb-errors-dev libxcb-randr0-dev
```

This installs package names, **not necessarily sufficiently new versions**. `debhelper` must provide compatibility level 13. Check the following version requirements before building.

### Debian 13 version gaps

The pinned recipes require newer versions of several libraries than Debian 13 stable provides. The versions below were observed in the build workstation's configured stable/backports repositories; availability can change.

| Dependency | Required by the recipes | Observed Debian 13 stable | Observed backports | Action |
|---|---|---|---|---|
| Meson | ≥ 1.3 | 1.7.0 | Not needed | Use the system package |
| wayland-protocols | ≥ 1.47 | 1.44 | 1.47 | Use a compatible backport |
| Wayland development files | ≥ 1.24 | 1.23.1 | No newer candidate observed | Obtain or build compatible Debian packages |
| libdrm development files | ≥ 2.4.129 | 2.4.124 | No newer candidate observed | Obtain or build compatible Debian packages |
| xkbcommon development files | ≥ 1.8 | 1.7.0 | 1.13.1 | Use a compatible backport |
| pixman development files | ≥ 0.46 | 0.44.0 | No newer candidate observed | Obtain or build compatible Debian packages |
| libinput development files | ≥ 1.27.1 | 1.28.1 | Not needed | Use the system package |

If `trixie-backports` is already configured and offers these versions:

```sh
sudo apt install -t trixie-backports wayland-protocols libxkbcommon-dev
```

For remaining gaps, prefer trusted Debian 13-compatible binary packages or rebuild/backport the relevant source packages inside the build environment. Install matching runtime and development packages together. Avoid upgrading the entire system to sid or using untracked `/usr/local` libraries to bypass Debian dependency checks.

Inspect your own candidates and check the first component:

```sh
apt-cache policy meson wayland-protocols libwayland-dev libdrm-dev \
  libxkbcommon-dev libpixman-1-dev libinput-dev
(cd wlroots && dpkg-checkbuilddeps)
```

**Stop if dependency checks fail.** Do not use `dpkg-buildpackage -d` to suppress missing or too-old dependencies. A successful private-prefix compile does not prove that Debian packaging prerequisites are met.

## Build Debian packages

Each submodule contains its own `debian/` packaging. There is no single root-level package build. Run the following from the repository root **inside the prepared build environment**.

Build each component, then install its generated runtime and development packages into that environment before building the next component:

```sh
# 1. wlroots
(cd wlroots && dpkg-checkbuilddeps && dpkg-buildpackage -b -us -uc)
sudo apt install ./libwlroots-0.20_*.deb ./libwlroots-0.20-dev_*.deb

# 2. SceneFX
(cd scenefx && dpkg-checkbuilddeps && dpkg-buildpackage -b -us -uc)
sudo apt install ./libscenefx-0.5-0_*.deb ./libscenefx-0.5-dev_*.deb

# 3. Anland producer
(cd anland && dpkg-checkbuilddeps && dpkg-buildpackage -b -us -uc)
sudo apt install ./libdisplay-producer5_*.deb ./libdisplay-producer-dev_*.deb

# 4. Mango
(cd mango && dpkg-checkbuilddeps && dpkg-buildpackage -b -us -uc)
```

`dpkg-buildpackage` writes artifacts to the parent directory, so the `.deb` files appear at the repository root. These globs assume that directory contains only the intended package versions and target architecture; otherwise pass the exact filenames. Stop after any failed build rather than continuing with old artifacts.

The recipes select `/usr` and the Debian multiarch library directory. **Do not run `sudo meson install` or `sudo cmake --install` as part of this workflow**: those commands install files directly and are not a substitute for building managed Debian packages.

On the target machine, install the four runtime packages together using their exact filenames. A runtime-only target does not need the three `-dev` packages, compilers, or build tools. APT resolves their ordinary runtime dependencies. The Mango package contains the launcher, user service, session files, configuration, and volume helper.

### What these packages do not supply

- A complete Android Anland installation: the Debian Anland recipe packages the **producer library and headers**, not the daemon or Android consumer.
- A device-specific GPU driver stack. Install Mesa appropriate for the GPU and ensure a working DRM render node.
- A desktop bar: `mangobar` is recommended, not a required build dependency. The session examples below assume it is installed; otherwise choose your own startup command.
- A running audio/session environment. PipeWire and the relevant user services still need to be configured for the target.

## Run a session

Before starting Mango, ensure the Android consumer and Anland daemon are ready, the daemon socket is available, and the chosen render node is accessible. On the reported Qualcomm/Adreno setup, use GPU-matched Mesa/Freedreno OpenGL. Zink requires a compatible Vulkan driver; it is not a universal fallback.

> Starting another producer can replace the producer already connected to the daemon. Do not run smoke tests or a second Mango instance against an active production socket.

### Manual startup

```sh
export ANLAND_SOCKET=/run/display.sock
export ANLAND_DRM_DEVICE=/dev/dri/renderD128
export XDG_RUNTIME_DIR=/run/user/$(id -u)
exec mango -s 'mangobar'
```

Use the real session runtime directory; it must exist and belong to the user. Adjust the socket and render node for your environment. `ANLAND_SOCKET` opts into the Anland backend; without it Mango uses its normal backend selection.

### Installed user service

Choose the service instead of starting a second manual instance:

```sh
systemctl --user enable --now mango-anland.service
mango-anland status
```

The launcher supports `start`, `stop`, `restart`, `kill`, and `status`. Volume control is provided by:

```sh
anland-volume.sh get
anland-volume.sh set 80
anland-volume.sh toggle
```

Other volume commands are `up` and `down`; `set` accepts `0–150`. State is stored in `$XDG_RUNTIME_DIR/anland-volume-state`, initially `100 0`.

## Verification and known limits

Keep these claims separate:

- **Historical device report:** the setup at the top and an earlier process-liveness smoke describe the previous snapshot, not independent hardware verification of these lifecycle fixes.
- **Current pinned fixes:** private builds and two CPU regression tests passed, including a UBSan run. An isolated daemon exercise observed slot `0 → 1 → 0`, ACK progression, and exact release. Codex's third independent source review found no blocking issues; review is not runtime proof.
- **Debian artifacts:** a full four-component `.deb` build has not been verified here. The latest prerequisite check still reported missing compatibility-level-13 tooling and too-old packaged Wayland, libdrm, and pixman development files.
- **Hardware behavior:** real Wayland wire integration, GPU fence import/export, scanout, and static-desktop visual output are not established by CPU tests. A later isolated whole-Mango launch failed during Zink/EGL renderer initialization; it did not verify a working desktop.

After configuring a Mango source build with the Anland backend enabled, run the included regression suite:

```sh
meson test -C mango/build --print-errorlogs
```

A version-print command or successful compilation alone is not graphical verification. Validate the actual target session, input, clipboard, text, reconnect, and buffer/fence lifetimes before treating a new stack as usable.

## Repository policy and credits

This repository owns the manifest and documentation. Product changes belong in the component repositories. Commit and publish those changes first, then advance the manifest's submodule pointers after rebuilding and verifying the stack. A README-only update must not imply that unpublished component fixes are included.

Project layout:

- `wlroots/`, `scenefx/`, `anland/`, `mango/`: separately maintained source repositories and Debian recipes.
- `README.md`, `README_zh.md`: English and Chinese build/run guides.
- [`.omp/skills/anland-v5-porting/SKILL.md`](.omp/skills/anland-v5-porting/SKILL.md): project-local OMP porting skill, available as `skill://anland-v5-porting` in OMP sessions started here.
- `qq/`: local reference material, ignored by Git and not distributed with this repository.

Component branches:

- [wlroots — anland5](https://github.com/luochen88/wlroots/tree/anland5)
- [SceneFX — anland5](https://github.com/luochen88/scenefx/tree/anland5)
- [Anland — anland5](https://github.com/luochen88/anland/tree/anland5)
- [Mango — anland5](https://github.com/luochen88/mango/tree/anland5)

Thanks to the original [Mango](https://github.com/mangowm/mango) and [Anland](https://github.com/SuperTurtleDev/anland) projects.

Further reading:

- [Anland user guide](https://github.com/SuperTurtleDev/anland/blob/legacy/doc/UserManual/anland_guide.md)
- [Mango installation documentation](https://mangowm.github.io/docs/installation)
- [Droidspaces USB Manager notes](https://github.com/KDJCPM/Droidspaces-rootfs-KDE-builder/blob/main/README_english.md#droidspaces-usb-manager)
