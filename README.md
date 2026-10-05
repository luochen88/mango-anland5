# Mango Anland 5 stack

[中文](README_zh.md)

This repository pins the four public `anland5` branches that make up the Mango native Anland 5 stack. The components are included as Git submodules so a checkout records the exact commits known to build together.

## Components

| Build order | Submodule | Branch | Pinned commit | Purpose |
|-------------|-----------|--------|---------------|---------|
| 1 | [`wlroots`](wlroots/) | `anland5` | `d57826345bfc9e39694c7d2fb8ea106a1c8fd273` | wlroots 0.20.2 with the external swapchain API Mango uses for Anland consumer-owned buffers. |
| 2 | [`scenefx`](scenefx/) | `anland5` | `3e73e479160069b6444a6011057a6cd7403ac9f0` | SceneFX 0.5 built against the wlroots branch above, with GBM failure diagnostics preserved. |
| 3 | [`anland`](anland/) | `anland5` | `6803932abb6f4cb191a461775d688d93298e6e6d` | Anland V3 producer library packaged as `display-producer` version 5.0.0. |
| 4 | [`mango`](mango/) | `anland5` | `91acc518688a4db979d05771d34dffec2be6d744` | Mango 0.17.5 with the native Anland backend, session scripts, and runtime volume control. |

Upstream repositories:

- <https://github.com/luochen88/wlroots/tree/anland5>
- <https://github.com/luochen88/scenefx/tree/anland5>
- <https://github.com/luochen88/anland/tree/anland5>
- <https://github.com/luochen88/mango/tree/anland5>

## Clone

Clone with submodules in one step:

```sh
git clone --recurse-submodules https://github.com/luochen88/mango-anland5.git
cd mango-anland5
```

If the repository was cloned without submodules:

```sh
git submodule update --init --recursive
```

To move every submodule to the latest published `anland5` branch tip instead of the pinned commits:

```sh
git submodule update --remote --merge
```

Stage and commit the updated submodule pointers afterward if you want to make a new tested stack snapshot:

```sh
git add wlroots scenefx anland mango
git commit -m "chore: update Mango Anland 5 stack pins"
```

## Build and install

The install prefix is `/usr/local`, matching the published branch documentation. Build and install in dependency order.

### 1. wlroots

```sh
meson setup wlroots/build --prefix=/usr/local --buildtype=debugoptimized
meson compile -C wlroots/build
sudo meson install -C wlroots/build
```

### 2. SceneFX

```sh
meson setup scenefx/build --prefix=/usr/local --buildtype=debugoptimized
meson compile -C scenefx/build
sudo meson install -C scenefx/build
```

### 3. Anland producer library

```sh
cmake -S anland -B anland/build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local
cmake --build anland/build --parallel
sudo cmake --install anland/build
pkg-config --modversion display-producer
```

Expected `pkg-config --modversion display-producer` output:

```text
5.0.0
```

### 4. Mango

```sh
meson setup mango/build -Danland=enabled -Dxwayland=enabled --prefix=/usr/local --buildtype=debugoptimized
meson compile -C mango/build
sudo meson install -C mango/build
```

## Run Mango on Anland

The Anland backend is selected by `ANLAND_SOCKET`. The default session scripts expect the daemon socket at `/run/display.sock` and the render node at `/dev/dri/renderD128`.

Manual session:

```sh
export ANLAND_SOCKET=/run/display.sock
export ANLAND_DRM_DEVICE=/dev/dri/renderD128
export XDG_RUNTIME_DIR=/run/user/$(id -u)
exec mango -s 'mangobar'
```

Systemd user session after installation:

```sh
systemctl --user enable --now mango-anland.service
mango-anland {start|stop|restart|kill|status}
```

Runtime audio volume state is controlled by:

```sh
anland-volume.sh {get|up|down|toggle|set <0-150>}
```

The state file is `$XDG_RUNTIME_DIR/anland-volume-state`; default state is `100 0`.

## Verification used for this snapshot

The pinned commits were checked locally with:

```sh
meson compile -C wlroots/build-anland5
meson compile -C scenefx/build-anland5
cmake --build anland/build-anland5 --parallel "$(nproc)"
meson compile -C mango/build-anland5
```

Additional smoke checks:

```sh
(cd wlroots && git diff --check)
(cd scenefx && git diff --check)
(cd anland && git diff --check)
(cd mango && git diff --check)
sh -n mango/scripts/mango-anland mango/scripts/anland-volume.sh
```

`anland-volume.sh` with a temporary `XDG_RUNTIME_DIR` returned:

```text
get       -> 100 0
set 80    -> 80 0
toggle    -> 80 1
```

`mango/build-anland5/mango -v` returned:

```text
mango 0.17.5(3b119a23)
```

A runtime smoke with `/run/display.sock` and `/dev/dri/renderD128` present started Mango with the service-equivalent environment and kept it alive until timeout. Android-side visual output, touch, keyboard, pointer, clipboard, volume observation, and reconnect behavior require manual verification on the Android consumer surface.

## Repository policy

This repository is only the stack manifest and documentation. Product code changes live in the component repositories above. Update this repository by advancing submodule pointers after rebuilding and re-smoking the complete stack.
