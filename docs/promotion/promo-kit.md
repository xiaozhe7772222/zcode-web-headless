# 引流素材包（zcode-web-headless）

按“精准度 × 转化率”排序。建议顺序：**先给上游提 issue → 再发中文社区 → 最后英文社区**。
同一段文案不要原样跨社区复制，容易被判定为 spam。

---

## 1. 上游 issue（最精准，优先）

仓库：<https://github.com/zai-org/ZCode/issues/new>

**Title**

```
Web-only mode cannot create projects: project.host-offline / remote-project-required
```

**Body**

```markdown
### What happens

Running the web surface without the desktop app:

```bash
node bin/zcode.mjs --web --workspace /path/to/project --port 3030
```

any attempt to create a project fails:

- picking a folder → `project.host-offline` ("Failed to get system info")
- "Work outside a project" → send fails with "draft session creation failed"
- server RPC `project.readDirectory {hostId:"local"}` → `project.host-offline`
- `reserveCreatedTask` → `project.remote-project-required`

### Why

`createServerHostServices()` binds project storage to a Source Server identity:

```ts
const projectStorageHostId = remoteHostIdFromWorkspaceIdentity(
  buildServerRemoteWorkspaceIdentity({ serverId, workspacePath: "/" }),
);
```

while the web client always sends `hostId: "local"`. `createProjectService().route()` compares it to
`options.ownerHostId` and, on mismatch with no `resolveHost`, throws `project.host-offline`.
So the web surface can never own a project on the machine it is running on.

### Proposed fix

Add an opt-in `localProjectStorage` flag (54 lines, patch attached). When the server serves the web
bundle, project storage stays on the local owner; `ZCODE_WEB_LOCAL_STORAGE=0` keeps upstream behaviour.

Patch: https://github.com/xiaozhe7772222/zcode-web-headless/blob/main/patches/0001-web-local-project-storage.patch
Working build + docs: https://github.com/xiaozhe7772222/zcode-web-headless

Verified on Linux x86_64 and arm64 (qemu + node 24 arm64): project creation, file writes, shell
execution and model calls all work without Electron.
```

> 提交前把“patch attached”改成“patch linked”，避免 GitHub 拒绝 `.patch` 附件。

---

## 2. V2EX · 分享创造

**标题**

```
[分享] 让 ZCode 不装桌面端也能用：浏览器里建项目、跑命令
```

**正文**

```
ZCode 官方有桌面端 / Web / TUI 三种形态，但 Web 端单独跑时是废的：

  project.host-offline
  project.remote-project-required

原因是服务端把项目存储绑到了 Source Server 身份，浏览器又固定发 hostId:"local"，
路由把 local 当成另一台离线机器，于是建项目、开会话全挂。

我把这两处改了（两个文件 54 行），现在 zcode --web 可以直接建项目、写文件、跑 shell、调模型。

顺手验证了 arm64（qemu + arm64 Node 24）和安卓 Termux + proot 的跑法。

仓库（Apache-2.0，基于上游 v3.16.2）：
https://github.com/xiaozhe7772222/zcode-web-headless

免编译包在 Releases，解压后一条命令：
  node bin/zcode.mjs --web --workspace ~/projects/demo --port 3030

已知边界：远程 SSH/WSL/Docker 设备仍然要桌面端，这个补丁只解决本机 Web。
```

---

## 3. linux.do

**标题**

```
把 ZCode 的 Web 端从 Electron 里拆出来了
```

**正文**

```
ZCode 只跑 web 时建不了项目，报 host-offline / remote-project-required。
根因是 projectStorageHostId 取了 source server 身份，而浏览器发的是 local。

改了 createServerHostServices 和 entry-http 两个文件，加了 localProjectStorage 开关，
Web 模式下项目存储归本机。现在能直接建项目、跑命令、对话。

x86_64 和 arm64（qemu 验证）都跑通了，安卓 proot 的方案也写了文档。

https://github.com/xiaozhe7772222/zcode-web-headless

有在用 ZCode 的可以试一下，尤其是想在服务器 / NAS / 手机上跑 web 端的场景。
```

---

## 4. Show HN

**Title**

```
Show HN: Run ZCode's web UI without the Electron desktop app
```

**Body**

```
ZCode (Apache-2.0 coding agent harness) ships desktop, web and TUI surfaces. The web surface alone
is unusable: creating a project fails with project.host-offline, because the server binds project
storage to a "Source Server" identity while the browser sends hostId:"local".

I traced it to createServerHostServices() and added a localProjectStorage flag (2 files, 54 lines).
Now `zcode --web` can create projects, write files, run shell commands and talk to any
OpenAI-compatible endpoint, with no desktop process.

Also verified on arm64 (qemu + node 24 arm64) and wrote up an Android proot path.

Repo: https://github.com/xiaozhe7772222/zcode-web-headless
Happy to answer questions about the storage routing or the arm64 story.
```

---

## 5. Reddit · r/selfhosted

**Title**

```
ZCode's web UI now works without the desktop app (headless patch, Apache-2.0)
```

**Body**

```
If you tried `zcode --web` on a server you probably hit:

  project.host-offline
  project.remote-project-required

Cause: project storage is bound to a Source Server identity, browser sends hostId:"local",
router treats it as an offline remote machine.

Fix: localProjectStorage flag in createServerHostServices (2 files, 54 lines). Web mode keeps
projects on the local owner. Works with any OpenAI-compatible endpoint, ships a Dockerfile, and
there's an Android/Termux proot guide.

https://github.com/xiaozhe7772222/zcode-web-headless

Remote SSH/WSL/Docker hosts still need the desktop app — this only makes local web self-sufficient.
```

---

## 6. X / 微博（短帖，配首屏截图）

```
ZCode 的 Web 端单独跑是废的：建项目报 host-offline，发消息报 draft session failed。

根因：服务端把项目存储绑到 source server 身份，浏览器发的却是 hostId:"local"。

改了 2 个文件 54 行，现在 zcode --web 能直接建项目、跑命令、调模型。
x86_64 / arm64 / 安卓 proot 都验证过。

github.com/xiaozhe7772222/zcode-web-headless
```

---

## 发帖前检查

| 项 | 要求 |
|---|---|
| Release 可用 | `releases/latest/download/zcode-3.16.2-linux.tar.gz` 能直接下载 |
| 首屏截图 | 用 `docs/web-headless/project-bound.png`，能一眼看到项目 + 终端输出 |
| 不夸大 | 明确“只解决本机 Web，远程设备仍需桌面端” |
| 不复读 | 每个社区换开头、换人称，别 Ctrl+V |
| 回复速度 | 发帖后 2 小时内盯评论，前 10 条回复决定推荐权重 |
| 合规 | 不要贴 API key、内网地址、中转站 IP |

## 预期

- 上游 issue：曝光精准，但转化慢（维护者合并前不会有大量流量）
- V2EX / linux.do：24 小时内可见明显访问量，star 转化约 1–3%
- HN：波动大，标题里 "without Electron" 是主要钩子；上首页可带来 100+ star
- Reddit：长尾稳定，r/selfhosted 对“服务器上跑 GUI agent”接受度高
