# 定制分支说明（custom/browser-remote）

本分支基于 [flyibeat/workbuddy-remote](https://github.com/flyibeat/workbuddy-remote)，
目标是让**浏览器端完整可用**：新建任务、发送消息、切换会话、历史加载都不再报错。

上游项目本身已经能跑通，但在"从浏览器新建任务并发送第一条消息"这条路径上存在若干断点。
本分支逐一补齐，并把排障方法沉淀到 [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md)。

---

## 一、相对上游的改动

### `src/web/workbuddy-native.mjs`（主体改动，+1100 行）

| 能力 | 说明 |
|---|---|
| 首页提交看门狗 | 捕获 Enter/发送按钮 → 拿到会话 id → 等原生发送 → 1.6s 未发则自行发送 → 跳转新会话 |
| cwd 补齐 | create 时若 `cwd` 为空，按「人工指定 → 上次使用 → 最近工作区 → 默认根目录」四级回退 |
| 运行时预热 | create 后 300ms、兜底发送前各调一次 `ensureRuntime` |
| 切换会话兜底 | 点击侧边栏卡片时确保该会话运行时处于运行态（同会话 60s 内只触发一次） |
| 打开指定会话 | 支持 `?openConv=<id>`：遮罩 + 轮询侧边栏卡片并点击 |

**发送参数必须用消息块数组**，这是本分支最关键的一行：

```js
await handle.sendPrompt([{ type: "text", text: draft }]);
```

### `src/server/bridge-server.mjs`（+169 行）

- 打过补丁的 chunk 强制 `Cache-Control: max-age=0, must-revalidate`，ETag 改为内容哈希。
  （上游默认 `immutable`，会导致补丁一年不回源，是最容易误判的一层）
- 首页提交相关补丁：`ownsSubmissionSurface()` 恒 true、关闭阻塞式 prewarm 抢占、
  `setContextWindow` / `setThoughtLevel` 非阻塞化、`captureTaskResources` 1.2s 超时。
- 启动期 shim 语法自检：首次生成 shim 时执行 `new Function(shim)`，语法失败立刻告警。
- 锚点未命中告警：`missedAnchors` 会打印具体标签，WorkBuddy 升级后能第一时间发现。

### `src/bridge/runtime.mjs` / `src/server/access-auth.mjs`

- 后台自动重连 supervisor。
- 登录失败限流（避免爆破）。

### `tools/start-workbuddy-remote.ps1`

- 支持从 `workbuddy-remote.config.json` 读取 `passwordHash`，不再只依赖环境变量。

---

## 二、新电脑部署（可照抄执行）

### 1. 前置条件

- Windows 10/11
- 桌面版 WorkBuddy（需开启 CDP 调试端口 9333）
- Node.js 18+（推荐 22.x）

### 2. 拉代码

```bash
git clone -b custom/browser-remote <你的仓库地址> workbuddy-remote
cd workbuddy-remote
npm install
```

> 若无法访问自有仓库，也可用上游代码 + 本分支的 patch：
> `git clone https://github.com/flyibeat/workbuddy-remote.git && git am 0001-*.patch`

### 3. 配置

复制 `workbuddy-remote.config.json` 并按下表填写（**该文件含密码哈希，不要提交**）：

| 字段 | 本机示例 | 说明 |
|---|---|---|
| `workbuddyExePath` | `C:\Users\<你>\AppData\Local\Programs\WorkBuddy\WorkBuddy.exe` | 桌面端主程序 |
| `workbuddyUserDataDir` | `C:\Users\<你>\.workbuddy\app` | 桌面端用户数据目录 |
| `cdpPort` | `9333` | CDP 端口，需与 WorkBuddy 启动参数一致 |
| `bridgePort` | `8780` | 网页服务端口 |
| `listenHost` | `0.0.0.0` | 局域网访问填 `0.0.0.0`，仅本机填 `127.0.0.1` |
| `passwordHash` | `sha256:<salt>:<hash>` | 用 `node tools/workbuddy-password-hash.mjs` 生成 |
| `killWorkBuddyProcessesBeforeStart` | `true` | 启动前清理旧进程，避免 CDP 端口占用 |

### 4. 启动

```powershell
powershell -ExecutionPolicy Bypass -File .\tools\start-workbuddy-remote.ps1
```

或直接跑 bridge（适合已有 WorkBuddy 在运行的情况）：

```bash
node src/main.mjs --cdp-port 9333 --host 0.0.0.0 --port 8780 ^
  --user-data-dir "C:\Users\<你>\.workbuddy\app" ^
  --workbuddy-pid <PID> ^
  --password-hash "sha256:..."
```

### 5. 验证

```bash
curl -s http://127.0.0.1:8780/healthz                 # {"ok":true}
curl -sI http://127.0.0.1:8780/agent-manager/ | head  # 200
curl -s  http://127.0.0.1:8780/bridge/workbuddy-native-shim.js | wc -c   # 应 > 200000
```

浏览器打开 `http://<主机IP>:8780/` → 输入密码 → 新建任务 → 输入内容 → 发送，
应在 1~3 秒内进入新会话并出现气泡。

---

## 三、日常维护

### 改完代码怎么生效

1. 重启 bridge（改 `src/server/*` 必须重启；只改 shim 也要重启，因为 shim 由服务端拼装下发）
2. 浏览器 **普通 F5** 即可（缓存已改为 `must-revalidate`，不必强刷）

### WorkBuddy 升级后

升级会替换 `app.asar`，所有**字符串锚点补丁**可能失效。启动后检查日志里是否出现：

```
[WorkbuddyAdapter] home-creation 补丁锚点未命中: <标签>
```

出现即表示需要重新取锚点。锚点位置见 `src/server/bridge-server.mjs` 中 `applyAnchor` 调用处。

### 同步上游更新

```bash
git fetch upstream
git rebase upstream/main          # 冲突集中在 shim 与 bridge-server 的补丁段
```

### 出问题先看哪里

一律先读 [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md) 的「五分钟定位流程」，
按 P0 → P2 顺序排查，不要从业务代码开始猜。

---

## 四、已知限制

- 依赖 WorkBuddy 前端内部实现（chunk 字符串锚点、`wb` SDK 结构），**客户端升级可能失效**。
- 浏览器端是内存路由，切换会话靠点击侧边栏卡片实现，不能直接改 URL 跳转。
- 会话历史存放在 `~/.workbuddy/projects/<encoded-cwd>/<id>.jsonl`，删除会话请走界面，
  不要手工删文件（易残留 sessions 表孤儿记录，表现为"记录损坏 4010"）。
