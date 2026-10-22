# 在 Android 上跑 ZCode Web UI（Termux + proot）

本文说明如何在**未 root 的安卓手机**上，通过 Termux + `proot-distro` 运行 ZCode 的浏览器端，然后用手机浏览器访问 `http://127.0.0.1:3030`。

> 为什么是 proot 而不是 chroot：`chroot` 需要 root 权限，绝大多数手机拿不到；`proot` 用 ptrace 做用户态路径重定向，不需要 root，代价是性能损耗（约 2–5 倍）。`proot-distro` 是 Termux 里管理 rootfs 的现成工具。

## 前置条件

| 项 | 要求 |
|---|---|
| 手机 | arm64（2017 年后绝大多数机型） |
| 系统 | Android 9+，建议 8GB 内存以上 |
| 存储 | 至少 3GB 可用（rootfs + Node + 包体） |
| 网络 | 能访问 npm / Node 下载源，或提前在电脑上准备好包 |
| 浏览器 | Chrome / Edge / Firefox 均可 |

## 方案 A（推荐）：电脑构建，手机只跑

手机端跑 `pnpm install` 很容易 OOM（node_modules 约 2.9GB），更稳的做法是在电脑上构建好，把 tar 包传过去。

### 1. 电脑上构建

```bash
git clone https://github.com/xiaozhe7772222/zcode-web-headless.git
cd zcode-web-headless
export ELECTRON_SKIP_BINARY_DOWNLOAD=1
mkdir -p packages/shared/dist && printf 'export {};\n' > packages/shared/dist/index.js
pnpm install
pnpm build:bootstrap
pnpm build:zcode --base-url https://example.invalid/zcode/
# 产物：dist/zcode/releases/3.16.2/zcode-3.16.2.tar.gz
```

### 2. 传到手机

任选一种：

```bash
# USB / adb
adb push dist/zcode/releases/3.16.2/zcode-3.16.2.tar.gz /sdcard/Download/

# 或电脑起个临时 HTTP 服务，手机浏览器下载
python3 -m http.server 8000
```

### 3. Termux 安装 proot 环境

```bash
pkg update && pkg install -y proot-distro termux-api
proot-distro install ubuntu
termux-wake-lock          # 防止息屏被杀
```

### 4. 进入 Ubuntu 装 Node 24（arm64 / glibc）

```bash
proot-distro login ubuntu

apt update && apt install -y curl xz-utils git ripgrep ca-certificates python3
curl -fsSLO https://nodejs.org/dist/v24.14.0/node-v24.14.0-linux-arm64.tar.xz
tar -xf node-v24.14.0-linux-arm64.tar.xz -C /opt
ln -sf /opt/node-v24.14.0-linux-arm64/bin/node /usr/local/bin/node
ln -sf /opt/node-v24.14.0-linux-arm64/bin/npm  /usr/local/bin/npm
node -v      # 必须 v24.x
```

> 必须用 **linux-arm64、glibc** 的 Node。Termux 自带的 `node` 是 Android/Bionic 目标，ZCode 的 `node-pty` 预编译（glibc）加载不了；Alpine（musl）同理不行。

### 5. 解压并启动

```bash
# 手机存储通常已挂到 /sdcard
tar -xzf /sdcard/Download/zcode-3.16.2.tar.gz -C ~
mkdir -p ~/projects/demo

cd ~/zcode
ZCODE_WEB_LOCAL_STORAGE=1 \
node bin/zcode.mjs --web --workspace ~/projects/demo --port 3030 --no-open
```

看到 `http://127.0.0.1:3030` 后，切到手机浏览器打开即可。

- proot 与 Android 共享网络命名空间，所以浏览器直接访问 `127.0.0.1:3030` 就能通。
- 不要加 `--host 0.0.0.0`，在手机上没有必要，还会把带执行权限的服务暴露到局域网。

### 6. 配置模型

在网页里 `Set → Add provider → Create custom provider`，或用 `~/.zcode/v2/provider_config.json`，格式见 [README.zh-CN.md](README.zh-CN.md#模型配置)。

## 方案 B：直接在手机上构建（不推荐）

```bash
proot-distro login ubuntu
apt install -y git curl xz-utils python3 make g++ ripgrep
# 装 Node 24（同上）
git clone https://github.com/xiaozhe7772222/zcode-web-headless.git
cd zcode-web-headless
mkdir -p packages/shared/dist && printf 'export {};\n' > packages/shared/dist/index.js
export ELECTRON_SKIP_BINARY_DOWNLOAD=1
pnpm install          # 手机耗时可能 30–90 分钟，峰值内存 2GB+
pnpm build:bootstrap
pnpm build:zcode --base-url https://example.invalid/zcode/
```

风险：`pnpm install` 阶段容易因内存不足被系统杀掉；构建产物也比直接拷贝大。除非你没有电脑，否则走方案 A。

## 日常使用

```bash
# 每次启动
termux-wake-lock
proot-distro login ubuntu -- bash -lc 'cd ~/zcode && ZCODE_WEB_LOCAL_STORAGE=1 node bin/zcode.mjs --web --workspace ~/projects/demo --port 3030 --no-open'
```

停止：终端里 `Ctrl+C`；退出 proot：`exit`。

## 已知限制与排错

| 现象 | 原因 / 处理 |
|---|---|
| `Cannot find module` / 启动即崩 | Node 版本低于 24，或用了 Termux 原生 node。确认 `node -v` 与 `which node` |
| `node-pty` 加载失败 | rootfs 是 Alpine（musl）。换 Ubuntu/Debian |
| 浏览器打不开 127.0.0.1:3030 | 服务没起成功；先在 proot 内 `curl -I http://127.0.0.1:3030` 自测 |
| 项目目录选不了 `/sdcard` | Android FUSE 权限限制，把项目放在 `~/projects` 下 |
| 息屏后任务中断 | 未执行 `termux-wake-lock`；另可在系统设置里给 Termux 关闭电池优化 |
| 后台进程被系统杀 | Android 12+ 的 phantom process killer，可 `adb shell settings put global settings_enable_monitor_phantom_procs false`（需一次 adb） |
| 运行明显变慢 | proot 的 ptrace 开销，属预期；简单任务可用，大型构建不建议 |
| 磁盘占用大 | 包体约 400MB + Node 约 120MB + rootfs 约 300MB |

## 安全

- 服务具备文件读写与命令执行能力，只监听 `127.0.0.1`，不要改成 `0.0.0.0`。
- API key 明文存在 `~/.zcode/v2/provider_config.json`，手机丢失或共享设备时及时清理。
- proot 不是安全沙箱，agent 能访问该 rootfs 内的一切，建议只在 `~/projects` 下工作。

## 验证依据

本仓库的 Linux arm64 路径已在 `qemu-aarch64-static` + `node-v24.14.0-linux-arm64` 下实测：服务端返回 200、agent 正常拉起、完整任务 9 秒完成、`node-pty`/`ssh2`/`koffi` 均加载成功。Android 的 proot 环境与之一致（arm64 + glibc），差异仅在 proot 的 ptrace 开销与 Android 的后台限制。
