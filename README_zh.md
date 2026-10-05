# Mango Anland 5 技术栈

[English](README.md)

本仓库固定组成 Mango 原生 Anland 5 技术栈的 4 个公开 `anland5` 分支。各组件以 Git submodule 形式集成，因此一次检出会记录已知可一起构建的精确提交。

## 组件

| 构建顺序 | Submodule | 分支 | 固定提交 | 用途 |
|----------|-----------|------|----------|------|
| 1 | [`wlroots`](wlroots/) | `anland5` | `d57826345bfc9e39694c7d2fb8ea106a1c8fd273` | 基于 wlroots 0.20.2，加入 Mango Anland 后端用于消费者管理 buffer 的外部 swapchain API。 |
| 2 | [`scenefx`](scenefx/) | `anland5` | `3e73e479160069b6444a6011057a6cd7403ac9f0` | SceneFX 0.5，基于上面的 wlroots 分支构建，并保留 GBM 失败时的真实错误诊断。 |
| 3 | [`anland`](anland/) | `anland5` | `6803932abb6f4cb191a461775d688d93298e6e6d` | Anland V3 producer 库，安装为 `display-producer` 5.0.0。 |
| 4 | [`mango`](mango/) | `anland5` | `91acc518688a4db979d05771d34dffec2be6d744` | Mango 0.17.5，包含原生 Anland 后端、会话脚本和运行时音量控制。 |

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

安装前缀为 `/usr/local`，与各公开分支文档一致。必须按依赖顺序构建和安装。

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

### 3. Anland producer 库

```sh
cmake -S anland -B anland/build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local
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
meson setup mango/build -Danland=enabled -Dxwayland=enabled --prefix=/usr/local --buildtype=debugoptimized
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

固定提交已在本地通过以下命令检查：

```sh
meson compile -C wlroots/build-anland5
meson compile -C scenefx/build-anland5
cmake --build anland/build-anland5 --parallel "$(nproc)"
meson compile -C mango/build-anland5
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
get       -> 100 0
set 80    -> 80 0
toggle    -> 80 1
```

`mango/build-anland5/mango -v` 返回：

```text
mango 0.17.5(3b119a23)
```

当 `/run/display.sock` 和 `/dev/dri/renderD128` 存在时，使用等价于 service 的环境变量启动 Mango runtime smoke，进程保持运行直到 timeout。Android 端画面、触摸、键盘、指针、剪贴板、音量观察和断线重连行为需要在 Android consumer 画面上人工验证。

## 仓库策略

本仓库只保存技术栈 manifest 和文档。产品代码改动属于上面的组件仓库。更新本仓库时，应在重新构建并重新 smoke 完整技术栈后推进 submodule 指针。
