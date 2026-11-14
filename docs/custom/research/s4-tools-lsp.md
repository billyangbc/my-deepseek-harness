# S4 — 模型工具、LSP、Markdown/预览与官方包零散补丁

分析对象：本 fork（`custom/main`，HEAD `c5ac2488d9`）相对上游最新（`39163dd30f`，0.2.2-rc.3）在下列范围内的二次开发，以及它们与官方的耦合与插件化可行性。范围：`packages/browser/tool-browser`、`packages/lsp/lsp-editor`、`packages/lsp/lsp-stdio`、若干官方包的零散补丁、`packages/client/ui-primitives`、`packages/client/ui-tool`、`packages/client/web`、`apps/web/tests`、`vitest.config.ts`、`packages/test-support/client-runtime`。只读分析，未改动源码。

基线口径：

- 分叉点 `47f943859b`；fork 在其上 +214 commits，上游在其上 +14926 commits（`git rev-list --count`）。
- 跨库对比命令：`export GIT_ALTERNATE_OBJECT_DIRECTORIES=<upstream>/.git/objects && git diff 47f943859b 39163dd30f -- <path>`。
- 结论使用的三个耦合等级：**可直接树外插件** / **需上游 seam** / **只能上游修**。

## 0. 一个决定全局的前提：上游删掉了 `ctx.apiProxy`

| 事实 | 证据 |
|---|---|
| 分叉点存在 `packages/host/apiproxy`，本 fork 继续在其上扩展 `host.browser.*` 等 RPC | `git cat-file -e 47f943859b:packages/host/apiproxy` → 存在；`packages/host/apiproxy/src/api-proxy.ts` 新增 `browserList`…`browserSelectOption`（约 3980–4264 行） |
| 上游 0.2.2 已删除该包，改为 `packages/api/{gateway,remotes,session-controller,workspace-controller,terminal-controller,…}` + Typert Remote | `git log --diff-filter=D -- packages/host/apiproxy` → `4f00a8b82a refactor(api): remove ApiProxy package`；`ls packages/host` 上游无 `apiproxy`；`ls packages/api` 上游有 `*-controller`、无 `*browser*` |
| 上游 `grep apiProxy` 全仓 0 命中；本 fork `grep apiProxy` 命中 `packages/host/apiproxy/*`、`packages/client/connection/*`、`packages/test-support/client-runtime/*` 等 | 两侧 grep |

因此：本 fork 的「Host RPC 层 + 客户端 runtime」是一整片**建立在被上游删除的接线上**的自研产品层。本报告范围内所有「工具/LSP/工具箱」定制都挂在这片接线之上；任何「上游合并」的动作都不是 cherry-pick，而是把产品层搬到新的 Remote 平面。这一点决定了下面绝大多数结论只能是 **需上游 seam** 或 **保留 in-tree**。

---

## 一、Agent 工具定制（tool-browser）分析与上游重叠

### 1.1 实现方式（文件 → 符号）

| 文件 | 关键符号 | 作用 |
|---|---|---|
| `packages/browser/tool-browser/src/index.ts` | `name='tool-browser'`、`inject=['apiProxy','workspaceRegistry','tools','systemPrompt']`、`Config`（`snapshotMaxBytes`）、`apply(ctx, config)` | 函数式插件（`name`/`inject`/`Config`/`apply`，无 default export），符合 `packages/AGENTS.md` 的插件形态约定 |
| 同上 | `defineTool({ name:'browser_navigate' … })` ×7、`finalizeSnapshot`、`ctx.systemPrompt.section({name:'tool:browser', order:107})` | 注册 `browser_navigate/snapshot/click/type/scroll/select_option/tabs`，并注入一段固定 systemPrompt 引导 |
| `packages/browser/tool-browser/src/host-bridge.ts` | `browserRequest()`、`RpcId('tool-browser-N')`、`BrowserHostError`、`unwrapBrowserResponse()`、`resolveWorkspaceId()`、`resolveTabId()`、`requireAgent()` | 手工构造 RPC 信封（`{rpcId, payload}`）、工作区解析、Tab 解析（显式 → selected → 首个 → 新建 `about:blank`） |
| `packages/browser/tool-browser/src/render.ts` | `renderNavigateSummary/ClickSummary/TypeSummary/ScrollSummary/SelectSummary/TabsSummary`、`boundSnapshotTree()` | 纯函数式模型侧摘要；快照按 UTF-8 字节截断并追加 `[snapshot truncated at N UTF-8 bytes]` |
| `packages/browser/tool-browser/src/invariant.ts` | `apply` + 空 `install`（`No runtime invariant: …`） | 满足「每包必有 invariant companion」门禁，声明无独立生命周期流 |
| `packages/browser/tool-browser/tests/{tool-browser,integration}.spec.ts` | `createBrowserHostStub()`、`fakeAgent()`、`mountAgentLoopTestDependencies()` | 单元 + agent-loop 集成：断言 `tool/call` + `tool/result` 落日志、`browser_snapshot` 的 terminal 结果卡、`finalizeContent` 截断 |

### 1.2 调用链与「不 import Playwright」

`execute(args, exec)` → `requireAgent(exec.agent)` → `resolveWorkspaceId(ctx, agent)`（`ctx.workspaceRegistry.list()` 里按 `workspace.sessionIds.includes(agent.id)` 找） → `resolveTabId(ctx.apiProxy, …)` → `ctx.apiProxy.host.browser*({…}, exec.signal)` → `unwrapBrowserResponse()`（失败抛 `BrowserHostError`，Host 文案直接进工具错误）。

Playwright 只在 Host 侧：`packages/host/apiproxy/package.json:78 "playwright": "^1.49.0"`、`packages/host/apiproxy/src/browser-registry.ts`（`launchPersistentContext` 按 workspaceId 一池）、`apps/desktop-host-pkg/package.json:36`。`tool-browser` 的依赖只有 `dsh-tools`/`dsh-system-prompt`/`dsh-host-apiproxy`/`dsh-workspace`/`dsh-agent`/`dsh-llm`（+ dev 侧 `agent-loop-testkit`、`spill-local`、`spill-policy`）。

### 1.3 Render intent

- `browser_snapshot`：`presentResult` → `{ card:'terminal', title:'Accessibility snapshot', output: raw }`，`streaming` 关闭；
  `output.render` 返回 `value.tree` 纯文本，`finalizeContent: finalizeSnapshot` 在 spill 之前对**完整结果**施加字节上限。
- 其余 6 个工具：`presentCall` → `{ card:'generic', title: 一行摘要, kind:'execute' }`；`output.render` 再给一行文本。
- 与 ADR 一致：`docs/adr/0008-embedded-browser-client-and-tools.md`「Agent 工具 render intent：`browser_snapshot` 为
  `terminal`（可 spill）；其余 `browser_*` 为 `generic` 一行摘要」。

### 1.4 Session 日志

没有新增 `SessionEventMap` 成员，也没有新事件类型：每次调用只产生既有 `tool/call` + `tool/result`（由 `ctx.tools` 统一落盘）。
因此**不涉及 `SESSION_FORMAT_VERSION`**，也不存在「模型可见但不可重建」的输入。
`tests/integration.spec.ts` 用 `findEvent(log, 'tool/call')` / `'tool/result'` 直接断言这一点。
人类在工具箱里的导航/Cookie 变化不进 Session 日志（`README.md` 与 systemPrompt 文案都显式声明），这是产品取舍而非缺陷。

### 1.5 挂载点

- `apps/cli/config/agent-presets/standard/agent.cordis.yml:253`、`code/…:254`、`cordis/…:242`（行 `tool-browser`）。
- `packages/bundle/web-app/package.json:111`、`packages/bundle/desktop-app/package.json:97`、`tsconfig.host.json:234`。

### 1.6 与上游的重叠关系（关键：架构不是同一件事）

| 上游资产 | 形态 | 与 fork 的重叠 |
|---|---|---|
| `packages/browser-use/browser-use/src/index.ts`（`BrowserUseRegistry`、`ctx.browserUse`、`register(name)` 单一 provider 槽） | **能力 seam（只注册名字）**，provider 在 `packages/experimental/browser-use-*`（Playwright MCP / Chrome DevTools MCP / Stagehand 原生） | 上游把「浏览器工具」交给 provider（MCP 工具表由外部服务器拥有），浏览器资源按 **Session** 归属；fork 的浏览器资源按 **Workspace** 归属，且**人类 UI 与 Agent 共用同一 Tab**。二者产品语义相反 |
| `packages/client/ui-sidebar-browser/src/client/*`（`browser/`、`definition.tsx`、`pages.ts`） | 右侧栏 iframe/`<webview>` 浏览（README：「sandboxed HTTP(S) pages… never injects Electron or Node access」） | 只是「在侧栏看网页」的 UI 目的重叠；**没有**自动化、没有 Playwright、没有 `browser_*` 工具 |
| `packages/experimental/browser-use-runtime/src/{index,mcp,error}.ts` | MCP 客户端生命周期、per-Agent 初始化 | 无对应物：fork 不启 MCP，直连 Host Playwright |
| 上游 `packages/api/*` | 无 `*browser*` controller | fork 的 `host.browser.*`（约 20 个 RPC：list/createTab/closeTab/selectTab/navigate/goBack/goForward/reload/snapshot/click/type/scroll/selectOption/resizeViewport/sendPointer/sendKeyboard/watchScreencast/showWindow）在上游**没有等价物** |
| 上游工具命名 | `browser_navigate` 仅出现在 `packages/experimental/browser-use-playwright-mcp/tests/upstream.e2e.ts`（验证 MCP 服务器自带工具） | 名字撞车，语义不同：上游同名工具由 Playwright MCP 提供 |

结论：**这不是「官方已有等价能力」**，而是两条互斥的浏览器路线。fork 路线的独有价值是「人与 Agent 共享同一有头 Chromium / 同一组 Tab / 同一登录态」，代价是把浏览器塞进 Host 与工具箱；上游路线是「provider 拥有浏览器、按 Session 隔离」，对人类可视化天然缺位。

### 1.7 耦合等级与建议

- **耦合等级：需上游 seam。** 直接原因不是工具本身，而是 `ctx.apiProxy` 与 `host.browser.*` 全被上游删除（§0）。
  工具代码 453 行基本是纯 Consumer，可移植性最好的一块；真正不可移植的是它脚下那 20 个 RPC。
- **建议：保留 in-tree 并记录。** 若将来要开源成树外插件，必须做两件事：
  1. 把 `host.browser.*` 迁到上游的 Remote 平面（`packages/api/remotes` 的 Typert 契约），或自带一个 Host service + Remote；
  2. 把 `resolveWorkspaceId` 依赖的 `workspaceRegistry.list().sessionIds` 换成上游当前的工作区绑定 API（`packages/workspace/workspace` 仍存在，但 `packages/api/workspace-controller` 已接管绑定语义）。
- **门禁缺口（in-tree 已存在，上游不会接受）**：`packages/AGENTS.md` 要求产品可见插件有「非单测的 REAL-composition
  测试」。`tool-browser` 只有 `ctx.provide('apiProxy', …)` 的手搭 harness，没有 Loader 级 `cordis.yml` 跑通；上游接受前需补。
  另：`apps/web/tests` 无任何 `browser_*` 会话快照（`grep browser_ apps/web/tests` 为空），模型可见输出缺快照覆盖。

---

## 二、LSP 定制分析

### 2.1 新增 Service Definition：`packages/lsp/lsp-editor`

| 文件 | 关键符号 |
|---|---|
| `src/index.ts` | `class LspEditor extends Service`（`super(ctx,'lspEditor')`）、`registerProvider()`、`syncDocument()`、`closeDocument()`、`hoverDocument()`、`normalizeExtension()` |
| `src/types.ts` | `LspEditorService`、`LspEditorProvider`、`LspEditorSyncRequest/CloseRequest/HoverRequest`、`LspEditorProvider*Request`、`LspEditorDiagnostic`（`severity: 'error'|'warning'|'info'|'hint'`）、`LspEditorHover` |
| `src/invariant.ts` | 空 `install`（`No runtime invariant: provider routes are private registry state.`） |

形态与既有 `ctx.lsp` 刻意对齐：按 `finalExtension(filePath)` 路由、注册期做全量校验（重复 id、空扩展、重复扩展、跨 provider 扩展冲突）再一次性 `ctx.effect` 提交，失败回滚。`types.ts` 里大量「为 Monaco 归一化」的字段说明它服务的是编辑器，不是模型工具。**消费者不是 Agent**：`packages/host/apiproxy/src/api-proxy.ts:3690/3736/3777` 用 `ctx.get('lspEditor')` 实现 `lspSyncDocument` / `lspCloseDocument` / `lspHoverDocument`，客户端侧在 `packages/client/ui-file-editor/src/client/EditorSurface.tsx`（`lspSyncDocument`/`lspCloseDocument`/`lspHoverDocument` props）。

### 2.2 provider 侧：`packages/lsp/lsp-stdio` 的改动

| 文件 | 改动 | 关键符号 |
|---|---|---|
| `src/index.ts` | `inject` 增加 `'lspEditor'`（**硬依赖**）；`LocalLspProvider implements LspProvider, LspEditorProvider`；新增 `editorInstances`、`editorQueues`、`enqueueEditor()`、`editorInstanceFor()`、`resolveWorkspace()`；`disposeAll()` 同时排空 editor 实例与队列 | `LocalLspProvider`、`enqueueEditor`、`editorInstanceFor` |
| `src/index.ts` | 新配置 `diagnosticsWaitMs`（默认 1500，`assertTimer` 校验）、`optional`（无法解析可执行文件时跳过该 server 而非整个插件加载失败） | `LspLocalServerConfig`、`validateServerConfig` |
| `src/editor-instance.ts` | **新增 346 行**：持久打开文档（`didOpen`/`didChange`，不再 transient open）、诊断等待与 300ms settle、版本门控 | `EditorLspInstance`、`DIAGNOSTICS_SETTLE_MS=300`、`waitForDiagnostics()`、`ensureDocumentOpen()` |
| `src/diagnostics.ts` | **新增**：`publishDiagnostics` 归一化与版本策略 | `normalizePublishDiagnostics()`、`readPublishDiagnosticsVersion()`、`shouldApplyPublishedDiagnostics()`、`shouldUpdateDiagnosticsCache()` |
| `src/host.ts` | 新增 `resolveHostSourceUri()`：只解析路径与 URI、不读字节（持久文档模式不需要读源码） | `resolveHostSourceUri` |
| `src/connection.ts` | `LspConnection` 构造函数**新增第 4 个参数** `onNotification`（默认 `() => {}`），把原来「忽略服务端通知」的分支改为回调 | 原注释 `// A server→client notification (e.g. diagnostics, logs): ignored by this MVP host.` |
| `src/instance.ts` | 适配新构造签名，显式传 `() => {}` | `LspInstance` 构造 |
| `tests/{diagnostics,editor-instance-diagnostics,pyright-editor}.spec.ts` | 归一化单测、实例级诊断测试、**keyless 真 pyright e2e**（Python 加空行后取诊断） | `pyright-editor.spec.ts` |

### 2.3 上游 0.2.2 的 LSP 体系现状

- `packages/lsp/{lsp,lsp-stdio,tool-lsp}` 三个包仍在，`ctx.lsp` 仍是**四个导航操作**的封闭并集
  （`goToDefinition | findReferences | goToImplementation | hover`，见 `docs/subsystems/lsp.md`；
  `packages/lsp/lsp/src/index.ts` 只有 `registerProvider`/查询，无诊断）。
- `docs/subsystems/lsp.md` **只描述导航**，没有 editor/diagnostics seam；本 fork 的 `docs/subsystems/lsp.md` 相对分叉点**零改动**
  （`git diff --name-only 47f943859b HEAD -- docs/subsystems/lsp.md` 为空）——文档没跟上代码。
- `packages/lsp/lsp-stdio/src/connection.ts` 仍写着「ignored by this MVP host」，即**通知通道上游仍未打开**。
- 上游 `tool-lsp` 是模型侧消费者，与编辑器诊断无关。

也就是说：**这部分是新 capability seam，不是「上游已修/已有等价能力」**。它同时补了两个上游空位：① 服务端→客户端通知通道（`publishDiagnostics`）；② 持久文档（`didOpen`/`didChange`）而非 transient open。后者是 `lsp-stdio` README 明确写下的第一阶段限制（「transient-open… Documents close after each call, so the first version needs no `didChange`, content cache, or document LRU」）。

### 2.4 这是「新 capability seam」还是「可树外插件」

拆成三段看：

1. **Service Definition（`lsp-editor` 包本身）**：纯插件，**可直接树外插件**。它不 import 官方任何私有物，
   只依赖 `dsh-lsp` 的 `LspError`/`LspProviderId`/`LspPosition`/`LspRange` 与其公共类型。
2. **provider 半边（`lsp-stdio` 内的 editor 实现）**：**需上游 seam**。树外 provider 想复用官方 `LspConnection`
   拿不到诊断——通知分支在官方代码里被硬编码丢弃，构造函数第 4 参就是 fork 为了让位 writer 才加的。
   树外替代方案只能自带 framing/decoder（`lsp-stdio` 只导出 `LspConnection`、`MessageDecoder`、`encodeMessage`），
   等于复制 300+ 行传输层并再起一个语言服务器进程（同一 workspace 两套进程、两套配置）。
3. **consumer 半边（Host RPC + Monaco）**：**需上游 seam**（`ctx.apiProxy` 已删，§0）。

最小上游化路径（建议按此向上游提）：给 `LspConnection` 一个正式的 notification 订阅口（或给 `ctx.lsp` 加 `registerNotificationListener`），再把 `lsp-editor` 作为新 seam 定义 + `lsp-stdio` 作为其 provider 提交。

### 2.5 其他必须记录的问题

- **`inject` 硬依赖**：`lsp-stdio` 现在 `inject: ['fs','lsp','lspEditor','subprocess']`。若某部署只挂 `lsp-stdio` 不挂
  `lsp-editor`，`lsp-stdio` **加载失败**（不是降级）。fork 两个 bundle 都成对挂载
  （`packages/bundle/web-app/cordis.patch.yml:79-84`、`desktop-app/cordis.patch.yml:80-84`），但这是脆弱点：
  建议改 `ctx.get('lspEditor')` 可选注入，或把 editor 注册放进「存在则注册」的分支。
- **`optional` 是为缺可执行文件兜底**（`cordis.patch.yml:129/134` 的 `gopls`/`jdtls`）：上游行为是「一个 server 解析失败 →
  整插件加载失败」，fork 改成跳过。上游是否接受值得单独提；语义上它把「配置错误」变成了「静默降级」，与 AGENTS
  「Misconfiguration fails loud」有张力，仅对**可选语言服务器**成立。
- **文档/门禁缺口**：`packages/lsp/lsp-editor` **没有 README、没有 tests**（`ls packages/lsp/lsp-editor` 只有
  `package.json/src/tsconfig.json`）。违反「每包 README + `## Known Limitations and Deferred Work`」与
  per-file 100% 覆盖门禁；`diagnosticsWaitMs` 也**未写进 `packages/lsp/lsp-stdio/README.md`**（该 README 只加了 `optional` 一行）。

---

## 三、官方包零散补丁逐项表（文件 → 改动 → 上游状态 → 结论）

| # | 文件 / 符号 | 本 fork 改了什么 | 为什么（CUSTOM.md 记录） | 上游状态（0.2.2-rc.3） | 删掉会怎样 | 结论（耦合等级 + 建议） |
|---|---|---|---|---|---|---|
| 1 | `packages/subprocess/subprocess/src/index.ts` → `scrubbedParentEnv()` | 额外剥离 `NODE_OPTIONS`（并把单行条件拆成 4 行 early-continue） | 2026-09-01：`dev:desktop` 给 Electron Main 设 `NODE_OPTIONS=--import tsx/esm`，LSP 子进程继承后在 workspace 解析 tsx 失败并退出 | **仍未剥离**（上游同函数只剥 `SENSITIVE_ENV_PATTERN` + `DSH_*`，另有 proxy 回填）。上游 `packages/boot/app-boot/src/index.ts` 的 `BOOTSTRAP_NAMES` 已把 `NODE_OPTIONS` 列为「`.env` 不得设置」的启动名——即上游自己也认为它是启动钩子 | LSP/pyright 子进程在 dev:desktop 下必挂；任何继承 `NODE_OPTIONS` 启动器的子进程都受污染 | **可直接树外插件？否**（是官方包内部行为）→ **建议上游化**：与 `BOOTSTRAP_NAMES` 的口径一致化为「子进程基线也不转发 NODE_OPTIONS」。当前**保留 in-tree 并记录**，上游接受前不要删 |
| 2 | `packages/subprocess/subprocess-local/src/process-inspector.ts` → `PosixProcessInspector`、`LinuxProcessInspector.arch`；`terminal.ts` → `LocalTerminalHandle` | 构造函数参数属性（`private readonly x`）改为显式字段 + 赋值 | 2026-08-29：兼容 Node 类型剥离（strip-only）加载 terminal 依赖链（web e2e） | **仍是参数属性**（上游 `terminal.ts:85-88`、`process-inspector.ts:360-361` 原样） | 行为零变化（产出 JS 等价）。只有在「不经 TS 转换、由 Node 直接剥离类型」的加载路径下才恢复报错（Node ≥22.18 默认剥离，参数属性属 `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`） | **纯语法兼容补丁，无功能价值**。建议：先确认 fork 哪条加载链路需要 strip-only（CUSTOM.md 只写「web e2e」，本次静态阅读未找到显式 `--experimental-strip-types`）；若属 fork 自身启动方式问题，应在 fork 侧修启动方式而不是长期持有这个 diff。上游合入价值低 → **保留 in-tree、标注为合并摩擦点** |
| 3 | `packages/session/session-persistence/src/coordinator.ts` → `appendCore` / `appendLiveBatch` | ① `state.materialized` 时每次追加前 `backend.loadStored(id)` 比对 `stored.events.length !== state.cursor` 则抛 `append cursor mismatch`；② `fresh.length === 0` 直接 return 不写盘 | 2026-09-02：桌面 Host 冷恢复写 `session/end-seed` 后内存 cursor 与磁盘不同步，再发消息时 `agent/inbox/spliced` 复用 seq（`seq gap … expected 28664, got 28663`） | **该文件已不存在**：上游把 persistence 重写为 `SessionHandle` 契约（`packages/session/session-persistence/src/{handle,storage-contract,errors,revision}.ts`），游标来自「primed cursor」+ `assertContiguous()`，并有 `SessionOwnershipLostError` / 单写者语义 | seq 复用类损坏会复发；配套的 `scripts/repair-session-log.ts` 也失去意义 | **只能上游修（且不应照搬）**。代价必须写明：`loadStored` 在 JSONL 后端是 `findLog()`（跨 project 目录找文件）+ `readPrefix()`（**整份解码**），即每条事件追加前一次全量读，长会话退化为 O(n²)；本 fork 的 `tests/persistence.spec.ts` 把 `backend.loadAttempts` 从 2 改成 3 就是这一成本的直接证据。建议：迁移到上游 `SessionHandle` 后**不移植**；若仍要防同类故障，用 revision/mtime/size 的**廉价**一致性检查，而不是重读日志 |
| 4 | `scripts/repair-session-log.ts`（新，89 行） | 一次性修数据脚本：扫 zstd 帧 → `SessionLogScanner` 重算 → 只删「`session/end-seed` 后紧跟重复 seq 的 `agent/inbox/spliced`」→ 备份后重写 | 与 #3 同一事故的现场修复工具 | 依赖的私有内部已被上游改动：`packages/core/session/src/chunk-rows.ts`（`decodeStorageRecord`）**上游已无此文件**；`session-persistence-jsonl/src/{zstd,format}.ts` 的导出面也变了 | 无运行时影响（脚本不参与构建）；删掉只丢失「历史损坏日志的修复手段」 | **无需上游化，保留 in-tree 归档**。建议在脚本头注明「依赖已迁移的私有格式，仅对 0.2.x 前会话文件有效」，避免未来误用 |
| 5 | `packages/client/ui-primitives/package.json` | 新增依赖 `"mermaid": "11.16.0"` | Mermaid 围栏渲染（PR #37–38） | **无 mermaid 依赖**（`grep mermaid packages/client` 0 命中） | Markdown 里 ` ```mermaid ` 回到普通代码块；文件编辑器预览丢失图表 | 见 §4.1 → **需上游 seam（但对上游是「已有 seam，可直接树外插件」）**：上游当前版本已有 `MarkdownDelegate.renderCode`（见 4.1），Mermaid 可做成不碰官方源码的客户端插件 |
| 6 | `packages/client/ui-primitives/src/markdown/*`：`MermaidBlock.tsx`、`mermaid-load.ts`、`MermaidBlock.module.css`、`MediaFrame.module.css`、`ZoomPanLightbox.tsx/.module.css`、`ZoomableImage.tsx`、`MarkdownImage.tsx` | 新增约 700 行：Mermaid SVG 渲染（主题跟随 `data-ds-dark-theme`、`securityLevel`、渲染失败回退代码块）、通用 zoom/pan lightbox、图片可放大 | Mermaid 代码块 + 可缩放 lightbox；Markdown/图片共用 ZoomPanLightbox（PR #37–38） | 上游有 `ImageLightbox.tsx`（72 行，**无缩放**，只 close/Escape/portal/焦点恢复），图片路径已接 `fileImages` delegate、拖拽保存、本地图片语法 | 会话里的图片回到上游式「点击看原图、不能缩放」，Mermaid 消失；文件编辑器 Markdown 预览降级 | **需上游 seam（组件层）**：这不是插件能加的 UI——`<img>` 生成在 `render.tsx` 内部。建议两条路：① 把 zoom/pan 能力**贡献进上游 `ImageLightbox`**（现在是并行重复实现，a11y 机制重复）；② Mermaid 走 `renderCode` delegate（4.1）。当前**保留 in-tree 并记录** |
| 7 | `packages/client/ui-primitives/src/markdown/{render.tsx, MarkdownText.tsx, plain-text.ts, inline-code-value.ts}` | `fenceLanguage()` 抽函数 + `mermaid` 分支；`MarkdownRenderContext.mermaidLabels`/`mermaidSecurityLevel`；`isBlankInlineCodeValue()` 抑制空白 inline-code 芯片 | ① 渲染 Mermaid；② 模型常输出只有空白/格式字符的反引号对，渲染成空胶囊 | 上游 `render.tsx` 已被重构（`FenceBlocks.tsx`、`MarkdownDelegate.tsx`、`local-image-syntax.ts`、`windows-path-destinations.ts`、`diff-fence.ts`、`cjkFriendlyStrong.ts`），fork 改的行号与结构与上游**不再对应** | Mermaid 失配 + 空白芯片回归（纯噪声，不影响数据） | 空白芯片抑制是**小而独立的上游化候选**（行为等价、无依赖）；其余 is 需 seam。合并风险：**高冲突区**（上游已重写同一文件） |
| 8 | `packages/client/ui-primitives/src/index.ts` + `packages/client/web/src/seed.ts` | 导出 `grammarLoadCount/highlightLines/subscribeGrammarLoaded`、`ZoomableImage`、`useScrollRevealScrollbar`、`MermaidSecurityLevel` 等；`seed.ts` 里再显式 pin 三个 highlight 符号 | 注释：「Pin syntax-highlight exports for plugin bundles (Git diff preview)」 | 分叉点 `index.ts` **未导出** highlight 三个符号（本次 `git show 47f943859b:…/index.ts` 验证）；上游已另起 `code-highlighting.ts` 导出 `CODE_HIGHLIGHT_EXTENSIONS/languageForPath/useCodeHighlighter` | `ui-git` 的 `diff-syntax-highlight.ts` 会走 `typeof primitives.highlightLines === 'function'` 的降级分支 → diff 无高亮（不崩） | 显式 pin **是冗余的**：`import * as UiPrimitives` 的 namespace 已包含新增导出。建议删 pin 保留 index 导出；上游侧需重新对齐到 `code-highlighting.ts`。**保留 in-tree** |
| 9 | `packages/client/ui-primitives/src/Menu.tsx` | ① `offsetHeight === 0` 时用 `getBoundingClientRect().height` 兜底并 `requestAnimationFrame(place)` 重排；② `side === 'top'` 时按锚点上沿反推 y，放不下就设 `maxHeight` 裁剪 | 菜单在隐藏预渲染阶段量不到高度，顶部/底部会错位或被切 | **无这些改动**（上游 `Menu.tsx` 无 `clipMaxHeight`、无 rAF 重排） | 顶部展开的菜单（文件树右键、Git 右键在窗口下半部）会溢出/被裁；这是真实可用性缺陷 | **可直接树外插件？否** → **建议上游化（小补丁、纯 UI 正确性）**。当前保留 in-tree |
| 10 | `packages/client/ui-primitives/src/icons/index.tsx` → `IconDiscardOutline16` | 新增 16px discard 图标（VS Code 语义，非 refresh） | Git 面板「撤销更改」 | 无此图标 | `ui-git` 编译失败（符号缺失） | 图标是**叶子依赖**，随 `ui-git` 一起树外化即可；单看它是「可直接树外插件」（图标可定义在插件内），**建议降级为插件私有**，减少官方包 diff |
| 11 | `packages/client/ui-primitives/src/useScrollRevealScrollbar.ts`（+ CSS 使用者） | 新增 `SCROLL_REVEAL_LINGER_MS`、`attachScrollRevealScrollbar()`、`useScrollRevealScrollbar()` | 会话区/终端/文件树/Git 共用「滚动时显示滚动条」 | **无对应 hook**（`grep useScrollRevealScrollbar` 0） | 4 个自研面板 + 会话区滚动条交互退化（需各自实现） | **可直接树外插件？否**（是共享 hook）→ 但消费者全是 fork 自有包，建议**下沉到 fork 自己的 client 包**而非官方 `ui-primitives`；或上游化（通用性足够）。当前保留 |
| 12 | `packages/client/ui-tool/src/client/contract/slots.ts` → `ToolCallOwnerProps.openDetails?`；`ToolCallTree.tsx`；`selection-target.ts`（新）；`components/ToolRow.tsx`；7 个 `toolviews/*.tsx` | 把 `openDetails` 从 chat node owner 一路透传给每个工具行；`ToolRow` 对**不可展开行**加 `role=button/tabIndex/Enter/Space` 并点击即开详情；`BashRow` 键盘处理改为「可展开则展开，否则开详情」；`selectionTargetForCall()` 生成 `SelectionTarget` | 2026-08（PR #38 前后）：「Tool 行点击可跳转工具详情并保持面板存活」 | **seam 上游本来就有**：分叉点 `packages/client/ui-conversation/src/client/contract/slots.ts` 第 677 行已有 `openDetails: (target: SelectionTarget) => void`；但上游 0.2.2 **已无 `SelectionTarget`/`openDetails`**，改为 `ToolRow` 内联渲染 `ToolDetails`（`details` model + `ToolDetails`），即「不跳栏」路线 | 工具行点击不再跳工具箱；`apps/web/tests` 两个 `lifecycle-chrome` 快照与 5 个新 `details-segmented-tab` 快照都基于跳转语义 | **需上游 seam（且上游已反向选择另一条路）**。fork 的做法在上游架构下无处安放。建议：**保留 in-tree 并记录为「与上游 ToolRow 内联详情分叉」**，合并时这段必冲 |
| 13 | `packages/client/ui-tool/tests/*`（`terminal-card`、`tool-details-render`、`toolview-slot`、`assembly-surfaces`、`chat-code-subcalls`、`diff-card`、`read-card`、`search-card`、`web-card`） | 新增 `detailsPanelTestProps()` helper、断言 `layout.openDetails` 被调用 1 次、删除「关闭详情」按钮相关用例 | 配合 #12 与工具箱分段 Tab（另一 scope） | 上游 ToolRow/ToolDetails 结构不同 | 无运行时影响 | 测试随实现走；**保留** |
| 14 | `packages/client/web/src/boot.tsx`、`AppRoot.tsx`(+`.module.css`)、`modules/src/client/manifest.ts` | 新增 `__DSH_HOST_BOOT__` 失败门禁：Host boot 失败时先渲染「Host 启动失败 + 重试」再进插件 boot；`retryHostBootHandler()` 读 `globalThis.dsh.retryHostBoot` | 桌面壳 integrated 模式：Main 里 Host 起不来时不能白屏 | **无 `__DSH_HOST_BOOT__`**（0 命中） | 桌面 integrated 模式 Host 失败 → 空白页无提示 | **可直接树外插件？否**（改 shell 入口）→ 属桌面交付层，与上游 `ui-sidebar-*`/Electron 路线无关。**保留 in-tree**；若上游出现桌面交付再谈 seam |
| 15 | `packages/client/web/src/seed.ts` | 额外 pin `grammarLoadCount/highlightLines/subscribeGrammarLoaded` | 同 #8 | 上游 `seed.ts` **已重写**为 `getStaticModules(): readonly DshClientModuleManifest[]` + `compiledModule()/platform()`，与 fork 的「对象字面量 spread」不再同构 | 同 #8（高亮降级） | 合并冲突点；**保留**（去 pin 见 #8） |
| 16 | `vitest.config.ts` | 顶层与每个 project 加 `resolve.alias`：`monaco-editor` → `packages/client/ui-file-editor/tests/monaco-editor.stub.ts`，`@xterm/*`、`@xterm/xterm/css/xterm.css` → `packages/client/ui-terminal/tests/*.stub.ts` | jsdom 加载不了 Monaco 的 AMD/CSS 图与 xterm 的 CSS | 上游**没有** `ui-file-editor`/`ui-terminal` 包（只有 `ui-sidebar-terminal`，xterm 仅在其依赖里）；上游 `vitest.config.ts` 无这些 alias | fork 的 client 单测在 jsdom 下直接崩（无法实例化 Monaco/xterm） | **测试基础设施，随 fork 的两个 client 包存在**；**保留 in-tree**。注意它是全局 alias（对所有 project 生效），未来若上游引入 Monaco 会互相遮蔽 |
| 17 | `packages/test-support/client-runtime/src/workspaces.ts`（+`tests/runtime.client.spec.tsx`） | `TestWorkspaces` 从「只实现既有接口」扩到 ~800 行：`listWorkspaceEntries`、`gitStatus/gitWorkingTree/gitInit/gitDiffPreview/gitStage/gitUnstage/gitDiscard/gitCommit/gitPush`、`readFile/writeFile/deletePath/renamePath/movePath/createWorkspaceDirectory`、`lspSyncDocument/lspCloseDocument/lspHoverDocument`、`terminalProfiles/Spawn/Write/Resize/Kill/List/Stream`、`browserList/CreateTab/CloseTab/SelectTab/Navigate/…/WatchScreencast` | 为工具箱五段与编辑器提供录制式测试替身 | 上游 `TestWorkspaces` 面完全不同（无 git/terminal/browser/lsp 面） | fork 的 client 测试无法编译 | **测试替身跟随接口走**；上游化前提是 Host RPC 面先对齐（§0）。**保留 in-tree** |
| 18 | `apps/web/tests/details-segmented-tab.e2e.ts`（新，158 行）+ 5 个新快照 + `scaffold.ts` 归一化 | 新的 keyless 浏览器回归：五段 Tab 壳层、编辑器空态、Git not-a-repository 空态、终端默认态、浏览器选中态；`normalizeAria` 增加 `(- tablist "Terminal tab":\n  - tab ")[^"]+` → `{{shell-profile}}` | 工具箱分段 Tab（V5 前后）的装配级快照 | 上游 `apps/web/tests` 是另一套（无 `details-segmented-tab`、无 `{{shell-profile}}` 归一化） | 分段 Tab 壳层与空态失去快照保护；**环境相关的 shell profile 名**会随机漂移导致快照不稳 | 归一化**本身是上游化候选**（任何带终端 Tab 的交付都会遇到 shell 名漂移）；测试文件**保留 in-tree** |
| 19 | `apps/web/tests/snapshots/lifecycle-chrome/{hero,plan-active}.expected.md` | 各删一行 `- button "Close details"` | 工具箱不再有关闭按钮（改为 Tab 化常驻） | 上游仍有关闭按钮语义 | 快照与实现不一致 | 跟随 #12/工具箱改造，**保留** |
| 20 | `packages/lsp/lsp-editor/*`、`packages/lsp/lsp-stdio/*` | 见 §2 | 编辑器诊断（V1 文件编辑器） | 上游无 editor LSP、无通知通道 | 文件编辑器无红波浪线/hover；`lsp-stdio` 因 `inject` 硬依赖 `lspEditor` **加载失败** | §2.4：定义层**可直接树外插件**；provider/consumer **需上游 seam**。当前**保留 in-tree 并记录** |

---

## 四、Markdown/预览/交互增强（ui-primitives、ui-tool）与上游差异

### 4.1 Mermaid：上游已有正确的 seam，fork 用错了位置

- fork：`render.tsx` 里 `if (!context.streaming && lang === 'mermaid') return <MermaidBlock …/>`——**改官方渲染器内部**。
  `mermaid-load.ts` 同步 `import mermaid from 'mermaid'`（11.16.0，体积大），`startOnLoad:false, securityLevel:'strict',
  maxEdges:2000`，主题按 `data-ds-dark-theme` 取 token 颜色写入 `themeVariables`，渲染串行化（`renderQueue`），
  `MermaidBlock` 失败回退 `CodeBlock`，并可展开进 `ZoomPanLightbox`。
- 上游：**无 mermaid**，但当前版本的 `packages/client/ui-primitives/src/markdown/MarkdownDelegate.tsx` 提供围栏渲染委派：
  `MarkdownDelegate.renderCode?: (fence: MarkdownDelegateFenceInfo) => ReactNode`，`MarkdownDelegateFenceInfo`
  = `{ lang, code, streaming, meta? }`，配套 `MarkdownDelegateProvider` / `useMarkdownDelegate`，`ui-chat` 已是消费者。
- 结论：**Mermaid 可以做成树外客户端插件**（provider 里把 `lang==='mermaid'` 交给 MermaidBlock，其余回落到
  `FenceCodeBlock`/`FenceMathBlock`），无需 `ui-primitives` diff。注意两处硬约束：① `renderCode` 一旦提供就**接管所有围栏**
  （含 `diff`、`math`），插件必须完整回落；② 该 seam 在上游**存在但 fork 未同步**（fork 的 `render.tsx` 还是旧结构），
  所以「迁移到 delegate」本身要求先吸收上游重构。

### 4.2 Zoom / Lightbox：与上游 `ImageLightbox` 并行重复

- fork：`ZoomPanLightbox.tsx`（238 行，portal + Escape + 焦点恢复 + `ZOOM_STEP/.25`、`ZOOM_MIN/.25`、`ZOOM_MAX/4` + 拖拽 pan +
  按钮 `IconCloseOutline16/IconPlusOutline16/IconRefreshOutline16`）、`ZoomableImage.tsx`（缩略图 + 放大按钮 + loading 态）、
  `MarkdownImage.tsx`（薄封装，写死 `loading:'lazy', decoding:'async', referrerPolicy:'no-referrer'`）。
- 上游：`ImageLightbox.tsx`（72 行，仅原图预览；`ImageLightboxLabels{dialog,close}`；处理「portaled image menu owns Escape」）。
- 风险：两套 portal/焦点/Escape 机制并存；上游一旦改 `ImageLightbox` 的 a11y 语义，fork 不会跟随。
- 建议：**把 zoom/pan 作为参数化能力合回上游 `ImageLightbox`**（或 fork 的 `ZoomPanLightbox` 直接复用上游的 a11y 外壳），
  删除重复实现；`MarkdownImage` 在上游应由 `renderImage` 的委托点接管（上游已有 `fileImages` + `DraggableImage` 路径，
  fork 的 `renderImage()` 改动与上游同名函数已完全不同）。

### 4.3 空白 inline-code 抑制

`inline-code-value.ts` 的 `isBlankInlineCodeValue()`（正则含 ZWSP/ZWNJ/软连字符/BOM 等）+ `render.tsx` 返回 `null` + `plain-text.ts` 的 `inlineText()` 同规则——纯输出卫生，无依赖、无配置，**上游化候选**。上游 `plain-text.ts` 仍把 `inlineCode` 原样返回。

### 4.4 Menus 与滚动条

- `Menu.tsx`：见 §三 #9。行为上是「隐藏预渲染测不到高度」的补丁，属真实缺陷修复，**建议上游化**。
- `useScrollRevealScrollbar.ts`：消费者全是 fork 自有面板（`ui-terminal/xterm-scroll-reveal.ts`、`ui-file-editor/FileTreePane.tsx`、
  `ui-git/GitPanel.tsx`、`ui-conversation/ConversationRoot.tsx`）。放在官方 `ui-primitives` 增加 diff，但不放又要四处复制 →
  建议**下沉到 fork 的共享 client 包**（或作为独立小包树外发布）。

### 4.5 ui-tool：详情跳转 vs 上游内联详情

- fork 用**已存在的 seam**（`ChatNodeOwnerProps.openDetails`，分叉点即有此字段）把「工具行点击」接到工具箱：
  `ToolCallOwnerProps.openDetails?` → `ToolCallTree` 包装成 `() => openDetails(selectionTargetForCall(block, toolName))` →
  `ToolRow.onOpenDetails` / 各 `toolviews`；`selection-target.ts` 负责把 running（`'kind' in block` → `{turnSeq: block.seq}`）
  与 settled（`{turnSeq: block.turn, stepSeq: block.step}`）两种 block 归一为 `SelectionTarget`。
- 细节：`ToolRow` 只在 `!expandable && onOpenDetails` 时给整行 `role=button`；`BashRow` 的 `toggleFromKeyboard` 从
  「不可展开直接 return」改成「可展开则 toggle，否则 `activateRow()`」——**这是行为变更**，需要 a11y/键盘回归
  （已有 `toolview-slot`/`terminal-card` 用例覆盖 `layout.openDetails` 调用次数）。
- 上游 0.2.2 走反方向：`ToolRow` 接收 `details?: ToolDetailsModel` 并内联 `<ToolDetails>`（`ToolRow.tsx:83/186/213/411`），
  不再有 `SelectionTarget`、`openDetails`（契约 grep 为空）。**这是一次产品级分叉**：fork「工具行 → 工具箱” vs 上游「工具行内展开」。
  合并时无法两边都要，必须在 ADR 里明确选择。

---

## 五、插件化可行性与改造建议

### 5.1 总体判断

1. 本 fork 的工具/LSP/Markdown 定制**不是散点补丁**，而是「Host RPC + 客户端 runtime + 工具箱」一条产品线的表层。
   上游在 0.2.2 删掉 `ApiProxy` 并重写会话持久化，等于**换掉了这条产品线的地基**（§0）。
   因此「尽量树外」在 fork 当前架构下只对少数叶子成立。
2. 真正**可直接树外插件**的只有三处：`lsp-editor` 的 Service Definition、Mermaid fence（走上游 `MarkdownDelegate.renderCode`）、
   以及图标/滚动条这类纯叶子（应下沉到 fork 自有包）。
3. 真正**只能上游修**的两处：session-persistence 的 seq 一致性（文件已不存在，且不应照搬重读方案）、以及任何
   `packages/host/apiproxy` 内部的 seam（它已不是官方包）。
4. 其余**需上游 seam**：LSP 通知通道、`ctx.apiProxy` 的替代 Remote 平面、`ToolRow` 详情交互方向、`ImageLightbox` 能力合并。

### 5.2 逐项建议（采用上游 / 移植 / 保留 in-tree 并记录）

| 定制点 | 耦合等级 | 建议 |
|---|---|---|
| `tool-browser`（7 个 `browser_*`） | 需上游 seam（`ctx.apiProxy` 已删） | **保留 in-tree**；先补 REAL-composition 测试与 `browser_*` 会话快照，再谈开源。上游化前提：`host.browser.*` 迁到 Typert Remote |
| `lsp-editor` 服务定义 | 可直接树外插件 | **移植**（可独立发布）；补 README + 单测（当前 0 测试、无 README） |
| `lsp-stdio` editor provider | 需上游 seam（`LspConnection` 无通知口） | **保留 + 向上游提 notification 订阅口**；同时把 `inject:['lspEditor']` 改为可选（`ctx.get`），消除「漏挂即启动失败」 |
| `lsp-stdio.optional` | 可直接树外插件？否（官方包配置） | 单独向上游提（语义要写清：仅对可选语言服务器，其余仍 fail loud） |
| `lsp-stdio.diagnosticsWaitMs` | 同上 | 向上游提同时**补 README**（当前未记录） |
| `docs/subsystems/lsp.md` | 文档 | **补写 editor/diagnostics 章节**（现零改动，`doc-sync` 会暴露） |
| `scrubbedParentEnv` 剥 `NODE_OPTIONS` | 需上游修（官方包语义） | **建议上游化**；与 `BOOTSTRAP_NAMES` 口径统一 |
| `subprocess-local` 参数属性 | 无功能价值 | **保留**，但追根因（fork 的 strip-only 加载路径）；长期应消除该 diff |
| `session-persistence` seq 校验 | 只能上游修 | 迁移到上游 `SessionHandle` 时**不移植**；如需同类防护，改用廉价 revision 检查 |
| `scripts/repair-session-log.ts` | 一次性运维脚本 | **保留归档**，头注声明仅适用于旧格式 |
| Mermaid 渲染 | 需上游 seam（但上游已有 `renderCode`） | **移植**为树外/树内 client 插件，走 `MarkdownDelegate`，删除 `render.tsx`/`MarkdownText.tsx` 的 mermaid 分支；`mermaid` 依赖随插件走 |
| ZoomPanLightbox / ZoomableImage / MarkdownImage | 需上游 seam（组件层） | **移植并入上游 `ImageLightbox`**（去掉并行 a11y 实现）；或至少复用上游外壳 |
| 空白 inline-code 抑制 | 小补丁 | **上游化**（`isBlankInlineCodeValue`） |
| `Menu.tsx` 定位/clip 修复 | 小补丁 | **上游化** |
| `IconDiscardOutline16` | 叶子 | 下沉到 `ui-git` 插件内，减少官方 diff |
| `useScrollRevealScrollbar` | 叶子 | 下沉到 fork 自有共享包 |
| `highlightLines` 导出 + seed pin | 冗余 | 删 seed pin；导出保留，待上游 `code-highlighting.ts` 对齐 |
| `ui-tool` 工具行跳详情 | 需上游 seam（上游反向选择内联详情） | **保留 in-tree 并在 ADR 记录分叉**；合并前必须决策「跳栏 vs 内联」 |
| `client/web` Host boot 失败门禁 | 需上游 seam（shell 入口） | **保留 in-tree**（桌面交付专属） |
| `vitest.config.ts` monaco/xterm alias | 测试基础设施 | **保留**；注意全局 alias 可能遮蔽未来上游同名依赖 |
| `test-support/client-runtime` 扩展 | 测试替身 | **保留**；接口面必须先与上游 Remote 对齐才可能上游化 |
| `apps/web/tests` 分段 Tab e2e + `{{shell-profile}}` 归一化 | 测试 | e2e **保留**；`normalizeAria` 的 shell 名归一化可**上游化** |

### 5.3 给后续合并的具体提醒（按风险排序）

1. **`packages/host/apiproxy` 整体**：不是「官方文件补丁」，而是「官方已删除的包」。任何 rebase 都必须先决定：
   自建 `browser`/`git`/`terminal`/`file`/`lsp` Remote（对齐 `packages/api/{gateway,remotes}`）还是长期分叉。
2. **`packages/client/ui-primitives/src/markdown/*`**：上游已重写该目录（`FenceBlocks`/`MarkdownDelegate`/
   `local-image-syntax`/`diff-fence`），fork 的 `render.tsx` 与 `MarkdownText.tsx` 改动**逐行都会冲**。
   建议现在就先做「Mermaid 迁到 delegate」这件事，把 diff 从渲染器内部挪走。
3. **`packages/client/ui-tool`**：上游 `ToolRow` 有 `ToolDetails` 内联模型，fork 有 `openDetails` 透传模型；
   这两个不是「谁新谁旧」，是两种交互。合并前先在 ADR/PRD 里定调，否则每次合并都要手工重写 8 个 `toolviews`。
4. **`packages/session/session-persistence`**：上游文件已换名换契约，`coordinator.ts` 的补丁**不可 cherry-pick**，
   只能在新契约里重写语义；顺带把「per-append 全量 `loadStored`」这个 O(n²) 成本丢掉。
5. **门禁**：`lsp-editor`（无 README/无测试）、`tool-browser`（无 REAL-composition、无会话快照）、
   `docs/subsystems/lsp.md`（零改动）三处会在 `doc-sync`/`test:coverage` 下暴露；in-tree 可用，
   但任何「向上游提」的动作都会先被这些门禁挡住。
