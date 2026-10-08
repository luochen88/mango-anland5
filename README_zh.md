# Mango Anland 5 技术栈

[English](README.md)

通过 Anland 在 Android 上运行 Mango Wayland 合成器。本仓库是**下游集成清单**：通过四个 Git submodule 固定需要配套使用的合成器和库，不替代上游 Mango 或 Anland。

![Mango Anland 5 桌面与 fastfetch](Screenshot_2026-10-05-22-44-32-28_048d46691f7e6d293bbd6f982d3d2551.jpg)

> **实验性项目。** 大部分集成代码由 AI 辅助编写，请审查组件差异并自行构建。目前仅报告 Lenovo Xiaoxin Pad Pro GT、Debian 13 环境可用，功能包括触摸、鼠标、触控板、音响、麦克风、键盘及双向剪贴板；不保证其他设备或构建可用。

## 目录

- [哪些组件需要自行构建](#哪些组件需要自行构建)
- [克隆固定版本](#克隆固定版本)
- [安装构建依赖](#安装构建依赖)
- [构建 Debian 包](#构建-debian-包)
- [运行会话](#运行会话)
- [验证结果与限制](#验证结果与限制)
- [仓库组织与致谢](#仓库组织与致谢)

## 哪些组件需要自行构建

**四个项目组件都要从固定源码构建；通用工具和基础库优先通过 APT 安装，但版本必须满足打包声明。**

| 顺序 | 组件 | 固定提交 | Debian 运行时包 | Debian 开发包 | 源码包版本 |
|---|---|---|---|---|---|
| 1 | [wlroots](wlroots/) | `f2ff3e6c040f474ca71e8a5639997934f7a99142` | `libwlroots-0.20` | `libwlroots-0.20-dev` | `0.20.2-1anland5` |
| 2 | [SceneFX](scenefx/) | `612c23fa80106ae0968a417ac711f2e4e8b4c9ef` | `libscenefx-0.5-0` | `libscenefx-0.5-dev` | `0.5.0-1anland5` |
| 3 | [Anland](anland/) | `b5ffb976c2e263701be6286a4afeadef3633d92a` | `libdisplay-producer5` | `libdisplay-producer-dev` | `5.0.0-2anland5` |
| 4 | [Mango](mango/) | `a6b0606a7db5ebd15426aa7e95c6155006f7e675` | `mango-anland5` | — | `0.17.5-2anland5` |

- **wlroots** 含 Anland consumer 管理 buffer 所需的外部 swapchain API，不能用发行版普通 wlroots 替代。
- **SceneFX** 必须针对上面的 wlroots ABI 构建。
- **Anland** 提供共享 `display-producer` 库和公共头文件。
- **Mango** 必须启用原生 Anland 后端。

SceneFX 依赖 wlroots；Mango 依赖三个库。Anland 可以独立构建，按表中顺序操作最简单。

### 固定快照与 ABI 配套

当前固定提交已包含 Anland/Mango 呈现、输入生命周期修复和两项永久 CPU 回归测试。Anland 新增可写目标 API，SONAME 仍为 5；Mango 需要配套的 `5.0.0-2anland5` producer revision。

任意检出版本都应以实际 `debian/changelog` 和 `debian/control` 为版本及精确依赖依据。不能因为 pkg-config 都显示 `5.0.0`，就把新 Mango 包与旧 producer 包混用。

## 克隆固定版本

```sh
git clone --recurse-submodules https://github.com/luochen88/mango-anland5.git
cd mango-anland5
```

已有仓库但未初始化 submodule：

```sh
git submodule update --init --recursive
```

构建快照时保留记录的提交。更新到分支最新提交会形成另一套技术栈，应重新构建、验证后再发布 submodule 指针。

## 安装构建依赖

推荐使用干净的 **Debian 13 构建容器或 chroot**，架构与目标设备一致。报告的目标架构是 `arm64`；下文是本机构建，不是交叉编译。不要在正在使用的桌面会话中混装构建依赖。

### 工具与通用开发包

当前 Debian 配方使用 Meson/Ninja 构建 wlroots、SceneFX 和 Mango，CMake/Ninja 构建 Anland；启用 GLES2、Xwayland、libliftoff 和 XCB errors。

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

该命令只安装包名，**不保证候选版本足够新**。`debhelper` 需要提供 compatibility level 13，开始构建前还需检查下列版本要求。

### Debian 13 的版本缺口

当前配方要求的部分基础库高于 Debian 13 stable。下表版本来自构建工作站配置的 stable/backports 源；软件源可用版本可能变化。

| 依赖 | 配方最低要求 | 观察到的 Debian 13 stable | 观察到的 backports | 处理方式 |
|---|---|---|---|---|
| Meson | ≥ 1.3 | 1.7.0 | 不需要 | 使用系统包 |
| wayland-protocols | ≥ 1.47 | 1.44 | 1.47 | 使用兼容 backport |
| Wayland 开发文件 | ≥ 1.24 | 1.23.1 | 未观察到更新候选 | 准备或回移兼容 Debian 包 |
| libdrm 开发文件 | ≥ 2.4.129 | 2.4.124 | 未观察到更新候选 | 准备或回移兼容 Debian 包 |
| xkbcommon 开发文件 | ≥ 1.8 | 1.7.0 | 1.13.1 | 使用兼容 backport |
| pixman 开发文件 | ≥ 0.46 | 0.44.0 | 未观察到更新候选 | 准备或回移兼容 Debian 包 |
| libinput 开发文件 | ≥ 1.27.1 | 1.28.1 | 不需要 | 使用系统包 |

已配置 `trixie-backports` 且其中版本满足要求时：

```sh
sudo apt install -t trixie-backports wayland-protocols libxkbcommon-dev
```

其他缺口优先使用可信、兼容 Debian 13 的二进制包，或在构建环境内回移对应源码包。运行时库和开发包应配套安装。不要整体混装 sid，也不要用未受包管理器管理的 `/usr/local` 库绕过 Debian 依赖检查。

检查自己的软件源并验证第一个组件的依赖：

```sh
apt-cache policy meson wayland-protocols libwayland-dev libdrm-dev \
  libxkbcommon-dev libpixman-1-dev libinput-dev
(cd wlroots && dpkg-checkbuilddeps)
```

**依赖检查失败就停止。** 不使用 `dpkg-buildpackage -d` 跳过缺少或过旧的依赖。私有前缀编译成功，不代表 Debian 打包依赖已满足。

## 构建 Debian 包

每个 submodule 都有自己的 `debian/` 配方，根目录没有单一的总包构建。下列命令从仓库根目录执行，**仅在准备好的构建环境内使用**。

每构建一个前置组件，就把生成的运行时包和开发包安装到该环境，再构建下一组件：

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

`dpkg-buildpackage` 把产物写到组件的父目录，因此 `.deb` 文件出现在仓库根目录。示例通配符要求该目录仅存在所需版本和目标架构的包；否则传入精确文件名。构建失败后停止，不继续安装旧产物。

配方使用 `/usr` 和 Debian multiarch 库目录。**此流程不运行 `sudo meson install` 或 `sudo cmake --install`**：它们直接安装文件，不能替代受包管理器管理的 `.deb`。

目标设备上使用四个运行时包的精确文件名一起执行 `apt install`。仅运行桌面的设备不需要三个 `-dev` 包、编译器或构建工具，APT 会解析普通运行时依赖。Mango 包包含 launcher、用户 service、会话文件、配置和音量脚本。

### 这些包不提供什么

- 完整 Android Anland 环境：Debian Anland 配方只打包 **producer 库和头文件**，不交付 daemon 或 Android consumer。
- 设备专用 GPU 驱动：需要安装匹配 GPU 的 Mesa，并准备可用 DRM render node。
- 桌面 bar：`mangobar` 是推荐项，不是必要构建依赖。下文示例假定已安装，否则使用自己的启动命令。
- 已运行的音频和用户会话环境：目标设备仍需配置 PipeWire 及相关用户服务。

## 运行会话

先确认 Android consumer、Anland daemon 已准备好，daemon socket 可用，render node 可访问。报告的 Qualcomm/Adreno 环境使用匹配 GPU 的 Mesa/Freedreno OpenGL；Zink 需要兼容 Vulkan 驱动，不是通用兜底方案。

> 新 producer 连接可能替换 daemon 当前连接的 producer。不要针对生产 socket 运行 smoke 或第二个 Mango 实例。

### 手动启动

```sh
export ANLAND_SOCKET=/run/display.sock
export ANLAND_DRM_DEVICE=/dev/dri/renderD128
export XDG_RUNTIME_DIR=/run/user/$(id -u)
exec mango -s 'mangobar'
```

使用真实会话 runtime 目录，要求目录存在且属于当前用户；根据环境调整 socket 和 render node。`ANLAND_SOCKET` 显式选择 Anland 后端，未设置时 Mango 使用正常后端选择流程。

### 已安装的用户 service

使用 service 时不要另起一个手动实例：

```sh
systemctl --user enable --now mango-anland.service
mango-anland status
```

launcher 支持 `start`、`stop`、`restart`、`kill`、`status`。音量控制：

```sh
anland-volume.sh get
anland-volume.sh set 80
anland-volume.sh toggle
```

还支持 `up`、`down`，`set` 范围为 `0–150`。状态保存在 `$XDG_RUNTIME_DIR/anland-volume-state`，初始值为 `100 0`。

## 验证结果与限制

以下结论分开理解：

- **历史设备报告**：页首环境和早期进程存活 smoke 来自前一快照，不是当前生命周期修复的独立硬件验证。
- **当前固定修复**：私有构建及两项 CPU 回归通过，UBSan 运行通过；隔离 daemon 场景观察到槽位 `0 → 1 → 0`、ACK 推进及精确 release。Codex 第三轮独立源码审计未发现阻塞问题；审计不等于运行时证明。
- **Debian 产物**：尚未验证四组件完整 `.deb` 构建。最近依赖检查仍报告缺少 compatibility-level-13 工具，以及过旧的 Wayland、libdrm、pixman 开发包。
- **硬件行为**：CPU 测试不证明真实 Wayland wire 集成、GPU fence import/export、scanout 或静态桌面视觉结果。后续隔离 whole-Mango 启动在 Zink/EGL renderer 初始化失败，没有验证可用桌面。

配置好启用 Anland 后端的 Mango 源码构建目录后，执行仓库内的回归套件：

```sh
meson test -C mango/build --print-errorlogs
```

打印版本或编译成功不代表图形验证通过。使用新技术栈前，应实际验证目标会话、输入、剪贴板、文本、重连、buffer/fence 生命周期。

## 仓库组织与致谢

本仓库负责 manifest 与文档，产品改动属于组件仓库。先提交并发布组件代码，重新构建与验证，再推进主仓库 submodule 指针；不能只更新 README 却声称已包含未发布修复。

项目结构：

- `wlroots/`、`scenefx/`、`anland/`、`mango/`：独立维护的源码仓库与 Debian 配方。
- `README.md`、`README_zh.md`：英文、中文构建运行指南。
- [`.omp/skills/anland-v5-porting/SKILL.md`](.omp/skills/anland-v5-porting/SKILL.md)：项目本地 OMP 移植 skill，在此启动的 OMP 会话中可通过 `skill://anland-v5-porting` 使用。
- `qq/`：本地参考材料，已加入 Git 忽略规则，不随仓库分发。

组件分支：

- [wlroots — anland5](https://github.com/luochen88/wlroots/tree/anland5)
- [SceneFX — anland5](https://github.com/luochen88/scenefx/tree/anland5)
- [Anland — anland5](https://github.com/luochen88/anland/tree/anland5)
- [Mango — anland5](https://github.com/luochen88/mango/tree/anland5)

感谢原作 [Mango](https://github.com/mangowm/mango) 和 [Anland](https://github.com/SuperTurtleDev/anland)。

相关资料：

- [Anland 新手使用教程](https://github.com/SuperTurtleDev/anland/blob/legacy/doc/UserManual/anland_guide.md)
- [Mango 安装文档](https://mangowm.github.io/docs/installation)
- [Droidspaces USB Manager 说明](https://github.com/KDJCPM/Droidspaces-rootfs-KDE-builder/blob/main/README_english.md#droidspaces-usb-manager)
