# ZCode Web Headless

**让 [ZCode](https://github.com/zai-org/ZCode) 的浏览器端脱离 Electron 桌面端独立运行。**
只跑 `zcode --web`，也能建项目、读写文件、执行命令、和模型对话。

[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Upstream](https://img.shields.io/badge/upstream-zai--org%2FZCode-lightgrey)](https://github.com/zai-org/ZCode)

[English](README.md) · [详细说明](docs/web-headless/README.zh-CN.md) · [安卓部署](docs/web-headless/ANDROID.zh-CN.md)

![项目内任务在浏览器中执行](docs/web-headless/project-bound.png)

## 问题

ZCode 有桌面端 / Web / TUI 三种形态，但 Web 端单独跑时什么也建不了，报错只有两句：

```
project.host-offline               # 选文件夹时：“Failed to get system info”
project.remote-project-required    # 发消息时：“Failed to send / 草稿会话创建失败”
```

原因是服务端把项目存储绑定到了 **Source Server 身份**，而浏览器固定发送 `hostId: "local"`，
路由把 `local` 当成另一台离线设备。于是目录浏览、项目保存、会话预留全部落到一个连不上的 Host 上。

结果就是：不装桌面端 → 建不了项目 → 发不出消息。**这跟你的 API key 没有任何关系。**

## 改动

两个文件，54 行。服务端在提供 Web 静态资源时，把项目存储归到本机：

| 文件 | 改动 |
|---|---|
| `packages/services/src/node.ts` | `createServerHostServices()` 新增 `localProjectStorage`；开启时 `projectStorageHostId` 为 `undefined`（本机 owner） |
| `packages/server/src/entry-http.ts` | 存在 `ZCODE_WEB_STATIC_ROOT` 时默认开启；`ZCODE_WEB_LOCAL_STORAGE=0` 可恢复上游行为 |

补丁：[`patches/0001-web-local-project-storage.patch`](patches/0001-web-local-project-storage.patch)

## 快速开始（免编译）

```bash
# 1. 下载并解压（本仓库构建，版本 v3.16.2）
curl -L -o zcode.tar.gz https://github.com/xiaozhe7772222/zcode-web-headless/releases/latest/download/zcode-3.16.2-linux.tar.gz
mkdir -p ~/zcode && tar -xzf zcode.tar.gz -C ~

# 2. 启动（需要 Node.js >= 24）
cd ~/zcode
node bin/zcode.mjs --web --workspace ~/projects/demo --port 3030 --no-open

# 3. 浏览器打开 http://127.0.0.1:3030
```

首次建项目会弹 **Trust this folder?**，确认后 agent 才能读写该目录。

## 从源码构建

```bash
export ELECTRON_SKIP_BINARY_DOWNLOAD=1

# 上游打包缺陷：@zcode/shared 是纯源码包，SEA/TUI 打包却要求 dist/。
# 纯 Web 不用 TUI 资源，占位即可。
mkdir -p packages/shared/dist && printf 'export {};\n' > packages/shared/dist/index.js

pnpm install
pnpm build:bootstrap
pnpm build:zcode --base-url https://example.invalid/zcode/
```

## Docker

```bash
docker build -f Dockerfile.web -t zcode-web .
docker run --rm -p 3030:3030 -v "$PWD/project:/workspace" zcode-web
```

## 安卓（Termux + proot）

未 root 手机可跑。`chroot` 需要 root，实际拿不到，所以走 `proot`：

```bash
pkg install proot-distro termux-api
proot-distro install ubuntu     # 里面装 linux-arm64 / glibc 版 Node 24
termux-wake-lock
node ~/zcode/bin/zcode.mjs --web --workspace ~/projects/demo --port 3030 --no-open
```

Termux 原生 Node（Bionic）和 Alpine（musl）都加载不了 `node-pty`，必须用 glibc 发行版。
完整步骤与排错见 [docs/web-headless/ANDROID.zh-CN.md](docs/web-headless/ANDROID.zh-CN.md)。

## 模型配置

任何 OpenAI 兼容端点都可以。ZCode 全程流式调用，端点必须稳定支持 `stream: true`
（非流式聚合差的服务会卡住）。

`~/.zcode/v2/provider_config.json`：

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

界面上的 “API format” 下拉有时点不动、会存成 `anthropic-messages`，遇到就改 JSON。

## 兼容性

| 检查项 | 结果 |
|---|---|
| 上游版本 | ZCode `v3.16.2` |
| Linux x86_64 | ✅ 全程实测，完整任务可用 |
| Linux arm64 | ✅ `qemu-aarch64-static` + arm64 Node 24 实测 |
| arm64 原生模块 | ✅ `node-pty`、`ssh2`、`koffi` 均加载成功 |
| 安卓 proot | ⚠️ 已写文档，未上真机验证 |

只有 `ssh2` 的 crypto 绑定和 `cpu-features` 是安装时按宿主编译，且仅用于远程 SSH 连接，
本地 Web 模式不加载。arm64 上重跑 `pnpm install` 即可得到原生构建，或走 ssh2 的 JS 回退。

## 边界

- 远程设备 / SSH / WSL / Docker Host 仍需桌面端，本补丁只让**本机 Web** 自足。
- 未绑定项目的任务会落到 `~/.zcode/workspace/draft-<uuid>/`，这是上游的草稿工作区设计。
- 服务端具备文件读写与命令执行权限，只监听 `127.0.0.1`；`--host 0.0.0.0` 会自动生成访问令牌，
  不要把带令牌的链接外发。

## 归属与许可

上游：[zai-org/ZCode](https://github.com/zai-org/ZCode)（Apache-2.0），原始说明见
[README.upstream.md](README.upstream.md)。本仓库修改同样以 Apache-2.0 发布，
`LICENSE`、`NOTICE.md`、`THIRD-PARTY-NOTICES.md` 保持原样，与上游 `main` 的差异仅为上述两个文件。
