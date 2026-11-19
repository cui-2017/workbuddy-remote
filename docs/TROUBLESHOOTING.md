# 故障排查与关键节点（定制版维护手册）

本文件是本定制分支的核心维护资料。出现"新建任务没反应 / 会话空白 / 记录损坏"时，
按下面的**分层顺序**排查，不要从代码逻辑开始猜。

---

## 一、关键节点分层

把整条链路拆成 6 层，任何一层断掉，表现都可能是"点了发送什么都没发生"。
**排查必须按 P0 → P2 的顺序**，因为上层故障会掩盖下层故障。

| 级别 | 节点 | 断掉后的表现 | 一句话判据 |
|---|---|---|---|
| **P0** | 静态资源缓存 | 改了代码但行为完全不变 | 响应头是否为 `immutable` |
| **P0** | 发送参数格式 | 有会话、无内容、无报错 | 日志有 `sendPrompt.dispatch-failed` |
| **P1** | 会话 cwd | 会话建了，点开报"记录损坏 4010" | `create arg={"cwd":""}` |
| **P1** | 运行时启动 | 状态一直 idle，消息不执行 | SDK 日志无 `ensureRuntime` |
| **P1** | 会话选中状态 | 首条消息不启动 | `conversation-created select=false` |
| **P2** | 历史目录存在性 | 读到 `enoent`，界面空白 | `history-probe persisted=false` |

---

## 二、逐层说明

### P0-1 静态资源缓存（最容易白白排查几小时）

WorkBuddy 的第三方 chunk 由 asar 内的静态服务下发，默认带：

```
Cache-Control: public, max-age=31536000, immutable
```

`immutable` 意味着**浏览器一年内不会回源**。任何通过对 chunk 做字符串替换实现的补丁，
只要文件名不变，就永远到不了客户端 —— 服务端日志里补丁命中正常，浏览器里却毫无变化。

**判据**：`curl -I <chunk-url>` 看 `Cache-Control`。若含 `immutable`，先怀疑这一层。

**修复**：`src/server/bridge-server.mjs` 对打过补丁的资源强制改写为

```
Cache-Control: public, max-age=0, must-revalidate
ETag: "<内容哈希>"
```

改完只需普通刷新，不必强刷。

---

### P0-2 发送参数格式（本轮真正的元凶）

浏览器端会话句柄的 `sendPrompt` 第一个参数**必须是 content blocks 数组**：

```js
await handle.sendPrompt([{ type: "text", text: "你好" }]);   // 正确
await handle.sendPrompt("你好");                              // 必崩
```

传字符串时，SDK 内部在 `e.content.map(...)` 处抛 `TypeError`，而这个错误**不会冒泡到界面**，
界面只会显示"暂无对话记录"，上层再包装成"会话记录损坏 4010"。

**判据**（`~/.workbuddy/logs/<date>/sdk/conversations/<id>.log`）：

```
runtime-status:persisted {"status":"working"}      ← 状态先变了
sendPrompt.dispatch-failed {"reason":"e.content.map is not a function"}  ← 2ms 后失败
```

**极其重要的陷阱**：`state` 会在消息真正派发**之前**就变成 `working`。
因此"状态是 working" **不能**作为"消息已发出"的证据。判断是否发出，
只认两个硬证据：

1. `handle.userPrompts()` 返回的数组长度 > 0
2. 会话 jsonl 里存在 `type: "message"` 的记录

---

### P1-1 会话 cwd 为空

`wb:conversations:create` 若收到 `{ cwd: "" }`，daemon 无法计算 owner key，
会话会写进 `sessions` 表，但**不会产生 jsonl 文件**，之后每次读取必然报错。

**判据**（`~/.workbuddy/logs/daemon.log`）：

```
[MethodCh] begin wb:conversations:create arg={"cwd":""}
[LocalConversationHistorySource] history-missing fallbackReason:"shared-enoent-owner-key-unavailable"
```

> 注意：`shared-enoent-owner-key-unavailable` 这个名字有误导性。
> 它的真实含义是"按候选路径找不到文件"（`trySelectFromRoot` 全部落空后走到 `missing()`），
> **与 owner key 本身无关**。不要被这个名字带偏。

**修复**：在 shim 的 create 包装里补齐 cwd，按
「人工指定 → 上次使用 → 最近工作区 → 默认根目录」四级回退。

---

### P1-2 运行时未启动

消息 `sendPrompt` 只是**入队**，真正执行要本地 CLI 就绪。
若从未调用 `ensureRuntime`，消息永远停在队列里。

**判据**：对比两个会话的 SDK 日志，成功的有

```
lifecycle:created → ensureRuntime → connection:create → connection:state connected
```

失败的一个 `ensureRuntime` 都没有。

**修复**：create 后 300ms 主动踢一次；保底发送前再确保一次。

---

### P1-3 会话选中状态

首页提交时若 `select: false`，会话会被建成"未选中"，首条消息既不启动也不报错。

**修复**：补丁把 `ownsSubmissionSurface()` 恒置为 `true`，并打印 `[HomeOwns] owns=` 便于确认。

---

### P2-1 历史目录不存在

历史文件路径规则：

```
~/.workbuddy/projects/<encoded-cwd>/<conversation-id>.jsonl
```

`<encoded-cwd>` 的编码规则是 **`:\` 合并成一个 `-`**：

```
C:\Users\MSI\WorkBuddy\2026-11-19-19-40-51
→ C-Users-MSI-WorkBuddy-2026-11-19-19-40-51
```

容易写错成 `C--Users-...`（多一个连字符）。写错后目录不会被复用，写入直接 ENOENT。

**判据**：`history-probe` 日志里的 `mainPath` 与磁盘实际目录对不上。

---

## 三、踩坑清单（工程层面，与业务逻辑无关）

| # | 坑 | 现象 | 规避 |
|---|---|---|---|
| 1 | shim 是**反引号模板字符串** | 注释/正则里写反引号或 `\/` 会截断字符串 | shim 内一律不用反引号；路径判断用 `indexOf`，别写正则 |
| 2 | shim 语法错误会**整体失效** | 页面能开，但 `window.wb` 是 undefined | 服务端首次生成 shim 时做一次 `new Function(shim)` 语法自检 |
| 3 | `Start-Process` 环境变量重复 | 抛"已添加项，字典中的关键字 HTTPS_PROXY" | 启动前清理仅大小写不同的重复项（`HTTPS_PROXY` vs `https_proxy`） |
| 4 | detached spawn 逃不出进程树 | 进程启动即被杀，err.log 为空 | 用宿主提供的后台任务方式启动，或注册计划任务 |
| 5 | 第三方 chunk 的 immutable 缓存 | 补丁永远不生效 | 见 P0-1 |
| 6 | 空壳会话 | 列表里有条目，点开报 4010 | 软删；不要手工造 jsonl 元数据除非确认格式 |
| 7 | 只看 `state` 判断是否发出 | 误判"已发送"，实际空会话 | 只认 `userPrompts().length > 0` 或 jsonl 里的 message |

---

## 四、五分钟定位流程

```
1. 界面报"记录损坏 4010"
   └─ grep "history-probe" daemon.log | tail
      ├─ persisted=false, reason=enoent → 目录/文件缺失（P2-1）
      └─ 有文件但报 e.content.map → 记录内容损坏

2. 发送后无反应、无气泡
   └─ grep "sendPrompt.dispatch-failed" sdk/conversations/<id>.log
      ├─ 有 → 参数格式错（P0-2）
      └─ 无 → 继续看 ensureRuntime 是否存在（P1-2）

3. 改了代码但行为没变
   └─ curl -I <资源url> | grep -i cache-control
      └─ 含 immutable → P0-1
```
