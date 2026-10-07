# Mango Anland 5 技术栈

[English](README.md)

本仓库固定组成 Mango 原生 Anland 5 技术栈的 4 个公开 `anland5` 分支。各组件以 Git submodule 形式集成，因此一次检出会记录已知可一起构建的精确提交。

![Mango Anland 5 桌面和 fastfetch](Screenshot_2026-10-05-22-44-32-28_048d46691f7e6d293bbd6f982d3d2551.jpg)

## 致谢和范围

感谢原作 [Mango](https://github.com/mangowm/mango) 和 [Anland](https://github.com/SuperTurtleDev/anland) 项目。本仓库只是用于通过 Anland 运行 Mango 的下游集成快照，不替代任何上游项目。

这些 `anland5` 分支中的大部分集成代码由 AI 辅助编写。请把它当作实验性集成代码处理：自行审查 diff、本地重新构建，并以各组件仓库为事实来源。

不保证可复现。本案例目前只报告在 Lenovo Xiaoxin Pad Pro GT、Debian 13 系统上实验通过。报告显示可正常使用的设备/功能包括：触摸、鼠标、触控板、音响、麦克风、键盘和双向剪贴板。

构建或运行前建议先阅读：

- [Anland 新手使用教程](https://github.com/SuperTurtleDev/anland/blob/legacy/doc/UserManual/anland_guide.md)
- [Mango 文档](https://mangowm.github.io/docs/installation)
- [Droidspaces USB Manager 说明](https://github.com/KDJCPM/Droidspaces-rootfs-KDE-builder/blob/main/README_english.md#droidspaces-usb-manager)

测试前请安装与你设备 GPU 匹配的 Mesa 驱动栈。在已测试的 Qualcomm/Adreno 环境中，请使用带 GPU 匹配 Freedreno OpenGL 驱动的 Mesa 构建；只有存在兼容 Vulkan 驱动时才使用 Zink，因为 Zink 是基于 Vulkan 实现 OpenGL。请让 Mango/Anland 指向可工作的 render node。

## 构建依赖

按下面的构建顺序操作前，请先安装所有 submodule 需要的构建工具和运行时开发包。不同发行版的软件包名不同；这个技术栈至少需要：

- C 编译器、C++ 编译器、`pkg-config`、`git`、`meson`、`ninja-build`、`cmake`
- Wayland、wayland-protocols、xkbcommon、pixman、libdrm、GBM/EGL/GLESv2
- libinput、udev/libudev、libseat、hwdata、libdisplay-info、libliftoff（可用时）
- Mango 需要 libpcre2-8、libcjson 和 pangocairo
- Anland 音频支持需要 PipeWire 开发文件
- 使用 `-Dxwayland=enabled` 构建 Mango 时需要 Xwayland、xcb、xcb-icccm 和 xcb-randr 开发文件

请安装与你 GPU 匹配的 Mesa 运行时和开发包。如果渲染器意外回落到软件渲染，应先修复 Mesa/DRM render node 配置，再判断 Mango 或 Anland 是否有问题。


## 组件

| 构建顺序 | Submodule | 分支 | 固定提交 | 用途 |
|----------|-----------|------|----------|------|
| 1 | [`wlroots`](wlroots/) | `anland5` | `f2ff3e6c040f474ca71e8a5639997934f7a99142` | 基于 wlroots 0.20.2，加入 Mango Anland 后端用于消费者管理 buffer 的外部 swapchain API，并打包为 `libwlroots-0.20`。 |
| 2 | [`scenefx`](scenefx/) | `anland5` | `612c23fa80106ae0968a417ac711f2e4e8b4c9ef` | SceneFX 0.5，基于上面的 wlroots 分支构建，并打包为 `libscenefx-0.5-0`。 |
| 3 | [`anland`](anland/) | `anland5` | `fe7b844dbde5eb91fac8af8b19f28e54f9cf1f0b` | Anland producer 库，通过 `display-producer` 5.0.0 导出 legacy facade。 |
| 4 | [`mango`](mango/) | `anland5` | `3ac767b9a707d7e14038be81fd616a95c50df395` | Mango 0.17.5，包含原生 Anland 后端、会话脚本、运行时音量控制和确定性的 Debian 包版本。 |

上游仓库：

- <https://github.com/luochen88/wlroots/tree/anland5>
- <https://github.com/luochen88/scenefx/tree/anland5>
- <https://github.com/luochen88/anland/tree/anland5>
- <https://github.com/luochen88/mango/tree/anland5>

## 克隆

一次性克隆仓库和 submodule：

```sh
git clone --recurse-submodules https://github.com/luochen88/mango-anland5.git
cd mango-anland5
```

如果克隆时没有拉取 submodule：

```sh
git submodule update --init --recursive
```

如果要把所有 submodule 从固定提交更新到已发布的 `anland5` 分支最新提交：

```sh
git submodule update --remote --merge
```

如果要生成新的已测试技术栈快照，更新后暂存并提交新的 submodule 指针：

```sh
git add wlroots scenefx anland mango
git commit -m "chore: update Mango Anland 5 stack pins"
```

## 构建和安装

必须按依赖顺序构建和安装。Debian 打包使用 `/usr` 和主机 multiarch libdir；手动本地安装可以使用 `/usr/local`，但每个组件的 `PKG_CONFIG_PATH` 和 `LD_LIBRARY_PATH` 必须指向同一个前缀。

### Debian 包元数据

每个 submodule 都包含固定 ABI 对应的 Debian packaging：

| Submodule | Source package | Runtime package | Development package |
|-----------|----------------|-----------------|---------------------|
| `wlroots` | `wlroots` | `libwlroots-0.20` | `libwlroots-0.20-dev` |
| `scenefx` | `scenefx` | `libscenefx-0.5-0` | `libscenefx-0.5-dev` |
| `anland` | `anland` | `libdisplay-producer5` | `libdisplay-producer-dev` |
| `mango` | `mango` | `mango-anland5` | — |

按相同顺序构建包。`scenefx` 依赖固定版本的 `libwlroots-0.20-dev`；`mango-anland5` 依赖固定版本的 wlroots、SceneFX 和 Anland 开发包。

### 1. wlroots

```sh
arch=$(dpkg-architecture -qDEB_HOST_MULTIARCH)
meson setup wlroots/build --prefix=/usr --libdir="lib/$arch" \
  -Dbackends=drm,libinput,x11 \
  -Drenderers=gles2 \
  -Dallocators=gbm,udmabuf \
  -Dsession=enabled \
  -Dxwayland=enabled \
  -Dexamples=false \
  -Dcolor-management=disabled \
  -Dlibliftoff=enabled \
  -Dxcb-errors=enabled
meson compile -C wlroots/build
sudo meson install -C wlroots/build
```

### 2. SceneFX

```sh
arch=$(dpkg-architecture -qDEB_HOST_MULTIARCH)
meson setup scenefx/build --prefix=/usr --libdir="lib/$arch" \
  -Dexamples=false \
  -Drenderers=gles2 \
  -Dtracy_enable=false \
  -Dcolor-management=disabled
meson compile -C scenefx/build
sudo meson install -C scenefx/build
```

### 3. Anland producer 库

```sh
arch=$(dpkg-architecture -qDEB_HOST_MULTIARCH)
cmake -S anland -B anland/build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr \
  -DCMAKE_INSTALL_LIBDIR="lib/$arch"
cmake --build anland/build --parallel
sudo cmake --install anland/build
pkg-config --modversion display-producer
```

`pkg-config --modversion display-producer` 的预期输出：

```text
5.0.0
```

### 4. Mango

```sh
arch=$(dpkg-architecture -qDEB_HOST_MULTIARCH)
meson setup mango/build --prefix=/usr --libdir="lib/$arch" \
  -Danland=enabled \
  -Dxwayland=enabled \
  -Dversion_suffix=release
meson compile -C mango/build
sudo meson install -C mango/build
```

## 在 Anland 上运行 Mango

通过 `ANLAND_SOCKET` 选择 Anland 后端。默认会话脚本预期 daemon socket 为 `/run/display.sock`，render node 为 `/dev/dri/renderD128`。

手动会话：

```sh
export ANLAND_SOCKET=/run/display.sock
export ANLAND_DRM_DEVICE=/dev/dri/renderD128
export XDG_RUNTIME_DIR=/run/user/$(id -u)
exec mango -s 'mangobar'
```

安装后的 systemd 用户会话：

```sh
systemctl --user enable --now mango-anland.service
mango-anland {start|stop|restart|kill|status}
```

运行时音频音量状态由以下脚本控制：

```sh
anland-volume.sh {get|up|down|toggle|set <0-150>}
```

状态文件为 `$XDG_RUNTIME_DIR/anland-volume-state`；默认状态是 `100 0`。

## 此快照使用的验证

固定提交已在本地通过 `/tmp/anland-stack-stage` 中的干净 staged install 检查：

```sh
meson setup /tmp/stage-wlroots wlroots --prefix=/usr --libdir=lib/$(dpkg-architecture -qDEB_HOST_MULTIARCH) -Dbackends=drm,libinput,x11 -Drenderers=gles2 -Dallocators=gbm,udmabuf -Dsession=enabled -Dxwayland=enabled -Dexamples=false -Dcolor-management=disabled -Dlibliftoff=enabled -Dxcb-errors=enabled
meson compile -C /tmp/stage-wlroots
DESTDIR=/tmp/anland-stack-stage meson install -C /tmp/stage-wlroots
meson setup /tmp/stage-scenefx scenefx --prefix=/usr --libdir=lib/$(dpkg-architecture -qDEB_HOST_MULTIARCH) -Dexamples=false -Drenderers=gles2 -Dtracy_enable=false -Dcolor-management=disabled
meson compile -C /tmp/stage-scenefx
DESTDIR=/tmp/anland-stack-stage meson install -C /tmp/stage-scenefx
cmake -S anland -B /tmp/stage-anland -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=lib/$(dpkg-architecture -qDEB_HOST_MULTIARCH)
cmake --build /tmp/stage-anland --parallel "$(nproc)"
DESTDIR=/tmp/anland-stack-stage cmake --install /tmp/stage-anland
meson setup /tmp/stage-mango mango --prefix=/usr --libdir=lib/$(dpkg-architecture -qDEB_HOST_MULTIARCH) -Danland=enabled -Dxwayland=enabled -Dversion_suffix=release
meson compile -C /tmp/stage-mango
DESTDIR=/tmp/anland-stack-stage meson install -C /tmp/stage-mango
```

额外 smoke 检查：

```sh
(cd wlroots && git diff --check)
(cd scenefx && git diff --check)
(cd anland && git diff --check)
(cd mango && git diff --check)
sh -n mango/scripts/mango-anland mango/scripts/anland-volume.sh
```

使用临时 `XDG_RUNTIME_DIR` 运行 `anland-volume.sh` 的结果：

```text
get -> 100 0
set 80 -> 80 0
toggle -> 80 1
```

`/tmp/anland-stack-stage/usr/bin/mango -v` 返回：

```text
mango 0.17.5(release)
```

当 `/run/display.sock` 和 `/dev/dri/renderD128` 存在时，使用等价于 service 的环境变量启动 Mango runtime smoke，进程保持运行直到 timeout。本地 smoke 日志只包含 Xwayland 显示 socket 已被占用警告和缺少 user bus 的消息。上文的 Lenovo Xiaoxin Pad Pro GT 报告显示 Android 端画面、触摸、鼠标、触控板、音响、麦克风、键盘和双向剪贴板在 Debian 13 上可用；这些 Android 端行为并未作为本仓库快照的一部分被独立重新测试。

`dpkg-checkbuilddeps` 能解析全部 4 个包。本工作站缺少 `debhelper-compat (= 13)` 和刚新增打包的固定版本开发包，所以没有在这里运行完整 `.deb` 构建。

## 仓库策略

本仓库只保存技术栈 manifest 和文档。产品代码改动属于上面的组件仓库。更新本仓库时，应在重新构建并重新 smoke 完整技术栈后推进 submodule 指针。
