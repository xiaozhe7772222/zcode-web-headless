# ZCode Web Headless

让 [ZCode](https://github.com/zai-org/ZCode) 的 **浏览器端脱离 Electron 桌面端独立运行** 的补丁与构建产物说明。

上游 ZCode 是 Apache-2.0 的 coding agent harness，提供桌面端 / Web / TUI 三种形态。但官方 Web 模式在没有桌面 Host 进程时无法新建项目、无法创建会话，浏览器里会报：

- `project.host-offline`（选择文件夹时 “Failed to get system info”）
- `project.remote-project-required`（发送消息时 “Failed to send / 草稿会话创建失败”）

本仓库修复了这个问题：**只跑 `zcode --web`，也能建项目、读写文件、执行命令、与模型对话。**

## 效果

| 项目内执行 `pwd && ls -la` | 创建并运行 `hello.py` |
|---|---|
| ![](docs/web-headless/project-bound.png) | ![](docs/web-headless/hello-run.png) |

arm64 实测（qemu + arm64 Node）：

![](docs/web-headless/arm64.png)

## 根因

Web 服务端把项目存储绑定到了 Source Server 身份：

```ts
const projectStorageHostId = remoteHostIdFromWorkspaceIdentity(
  buildServerRemoteWorkspaceIdentity({ serverId, workspacePath: "/" }),
);
```

而浏览器在 “这台电脑” 上创建项目时固定发送 `hostId: "local"`。服务端路由判定 `local !== ownerHostId`，把它当成一台远端设备：

```ts
if (!hostId || hostId === (options.ownerHostId ?? "local")) return null; // 本地
if (!options.resolveHost) throw new Error("project.host-offline");       // 远端
```

于是目录浏览、项目保存、会话预留全部落到一个永不离线的 Host 上。

## 改动

| 文件 | 内容 |
|---|---|
| `packages/services/src/node.ts` | `createServerHostServices` 新增 `localProjectStorage` 选项；开启时 `projectStorageHostId` 为 `undefined`，项目与任务存储归本机 owner |
| `packages/server/src/entry-http.ts` | 纯 Web 入口（存在 `ZCODE_WEB_STATIC_ROOT`）默认开启；`ZCODE_WEB_LOCAL_STORAGE=0` 可关闭，恢复 Source Server 语义 |

补丁：`patches/0001-web-local-project-storage.patch`

```bash
git apply patches/0001-web-local-project-storage.patch
```

## 构建

```bash
export ELECTRON_SKIP_BINARY_DOWNLOAD=1

# 上游已知问题：@zcode/shared 是纯源码包，没有 build 脚本，SEA/TUI 打包会因缺少 dist 中断。
# 纯 Web 用不到 TUI 资源，占位即可。
mkdir -p packages/shared/dist && printf 'export {};\n' > packages/shared/dist/index.js

pnpm install
pnpm build:bootstrap
pnpm build:zcode --base-url https://your-host/zcode/

tar -xzf dist/zcode/releases/*/zcode-*.tar.gz -C /opt
node /opt/zcode/bin/zcode.mjs --web --workspace /path/to/project --port 3030
```

## 运行

```bash
# 默认只监听 127.0.0.1
node bin/zcode.mjs --web --workspace /path/to/project --port 3030

# 对外提供服务（会自动生成访问令牌）
node bin/zcode.mjs --web --host 0.0.0.0 --port 3030
```

首次在网页里创建项目会弹出 **Trust this folder?**，确认后 agent 才能读写该目录。

## 安卓（Termux + proot）

未 root 的安卓手机也能跑：

```bash
pkg install proot-distro termux-api
proot-distro install ubuntu
proot-distro login ubuntu          # 在里面装 arm64 glibc 版 Node 24
termux-wake-lock
node ~/zcode/bin/zcode.mjs --web --workspace ~/projects/demo --port 3030 --no-open
```

手机浏览器打开 `http://127.0.0.1:3030` 即可。

- `chroot` 需要 root，手机上基本拿不到，所以走 `proot`（ptrace 路径重定向，性能损耗约 2–5 倍）。
- 必须用 **linux-arm64 + glibc** 的 Node；Termux 原生 Node（Bionic）和 Alpine（musl）都加载不了 `node-pty`。
- 建议电脑上构建好 tar 包再传手机，手机端 `pnpm install` 容易 OOM。

完整步骤、日常启动脚本、排错表见 [ANDROID.zh-CN.md](ANDROID.zh-CN.md)。

## 模型配置

支持任何 OpenAI 兼容端点。编辑 `~/.zcode/v2/provider_config.json`：

```json
{
  "schemaVersion": 1,
  "config": {
    "providerOrder": ["my-provider"],
    "providerConfigRules": {
      "providerRules": [{
        "providerId": "my-provider",
        "providerName": "My Provider",
        "config": {
          "group": "standard-personal",
          "access": { "type": "api-key", "apiKey": "<YOUR_API_KEY>" },
          "api": { "type": "openai-chat-completions", "baseUrl": "https://your-endpoint/v1" },
          "personalModelIds": ["glm-5.2"],
          "modelOrder": ["glm-5.2"]
        }
      }]
    },
    "modelConfigRules": {
      "providerModelRules": [
        { "modelId": "glm-5.2", "config": { "enabled": true }, "providerId": "my-provider" }
      ],
      "manualProviderModelRules": []
    }
  }
}
```

> 界面上的 “API format” 下拉有时点击不生效，会默认存成 `anthropic-messages`，需手工改成 `openai-chat-completions`。

验证要点：

- 中转站/服务需支持 **流式**（`stream: true`）。ZCode 全程流式调用，非流式聚合差的服务会卡住。
- 模型必须在该 key 的可用范围内，否则会重试 8 次后报 `403 model is not available in the current token plan`。

## 架构支持

| 组件 | 情况 |
|---|---|
| 服务端 / Agent / Web | 纯 JS，跨架构 |
| `node-pty` | 预编译含 linux/darwin/win 的 arm64 与 x64 |
| `koffi`、`unsafe-pointer` | 预编译含 arm64 |
| `ssh2` crypto、`cpu-features` | 安装时按宿主编译（仅 SSH 远程用，本地 Web 不加载） |

- **arm64 已验证**：`qemu-aarch64-static` + `node-v24.14.0-linux-arm64`，服务端 200、agent 拉起、完整任务 9 秒完成、`node-pty`/`ssh2`/`koffi` 均正常加载。
- qemu 下 `uname -m` 显示 `x86_64` 属模拟器不虚拟 uname，进程本身是 aarch64。
- 要出正式 arm64 包，直接在 arm64 机器/CI 上执行同样的构建命令即可，代码无需再改。
- Linux 需 **glibc**（node-pty 预编译为 glibc 版，Alpine/musl 不适用）。

## 容器

```bash
docker build -f Dockerfile.web -t zcode-web .
docker run --rm -p 3030:3030 -v "$PWD/project:/workspace" zcode-web
```

## 边界

- 远程设备 / SSH / WSL / Docker Host 仍需要桌面端 Host，本补丁只解决 **本机 Web**。
- 未绑定项目的任务会落在 `~/.zcode/workspace/draft-<uuid>/`，这是上游的草稿工作区设计。
- 服务端具备文件读写与命令执行权限，**不要**把端口直接暴露到公网而不加鉴权。

## 许可证与归属

- 上游：<https://github.com/zai-org/ZCode>，Apache-2.0。
- 本仓库保留上游 `LICENSE`、`NOTICE.md` 与 `THIRD-PARTY-NOTICES.md`，修改部分同样以 Apache-2.0 发布。
- 修改仅涉及上述两个文件（54 行），其余代码与上游 `main` 一致。
