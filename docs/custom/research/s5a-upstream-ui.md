# S5a：上游 0.2.2 与 fork 定制 UI 功能重叠盘点

- 上游：`<upstream 工作区>`，HEAD `39163dd30f`，版本 `0.2.2-rc.3`
- 本 fork：`<fork 工作区>`，`custom/main`
- 方法：只读代码与文档。上游路径均相对上游根；fork 路径均相对 fork 根。结论以 grep 证据为准（`monaco` / `CodeMirror` / `writeFile` / `sidebarRightTabs.register` / `diagnostic` / `mermaid`）。

## 一、映射总表

| # | fork 功能（源包） | 上游对应包 | 成熟度 | 行为差异 | 建议 |
|---|---|---|---|---|---|
| 1.1 | 文件树 `ui-file-editor`（懒加载、类型图标、Git 徽标） | `client/ui-sidebar-files` | 默认装载（`bundle/web-app`），成熟 | 上游同样逐级懒加载 + 目录直系 watch + 按 pane 记忆展开/宽度；无新建/重命名/删除/拖拽移动；树根固定为 Session cwd，不能越出 | 树交互（工作区搜索、隐藏项开关、Gitignore 开关）值得移植；fork 的写操作能力保留 |
| 1.2 | Monaco 多 Tab 编辑 + 显式保存 | **无对应**（全仓无 `monaco`/`CodeMirror`；`workspace-files` 明确「exposes no mutation operation」） | 上游无此能力 | 上游只有只读预览；写文件只在 Agent 侧：`fs/tool-fs` 的 `write`/`edit`、`fs/tool-str-replace-editor` 的 `str_replace_editor` | 保留 fork，是核心差异价值 |
| 1.3 | 多 Tab / 分栏（编辑器内自建 tab 池） | `client/ui-sidebar-right` + `client/ui-dockkit` | 默认装载，成熟 | 上游 tab = 资源地址，可拆分/浮动/持久化布局；每 tab 一个 `dsh-resource://` 地址 | 若长期重构，用上游 tab-type registry 承载编辑器，而不是自建 tab 池 |
| 1.4 | 图片预览 | `client/ui-sidebar-documentpreview` + `client/ui-primitives`（`ImagePreview`/`ImageLightbox`） | 默认装载，成熟 | 上游图片预览支持 fit-width、25%–400% 缩放、拖拽平移、主题背景；`ImageLightbox` 仅纯图放大（无缩放/平移） | 采用上游图片渲染与 `ZoomControls`，fork 仅保留 lightbox 全屏能力 |
| 1.5 | Markdown 渲染 / WYSIWYG | `client/ui-primitives`（`MarkdownText`） | 默认装载，成熟 | 上游 GFM + KaTeX + Shiki 流式增量高亮、frontmatter、`pathImages`、源码行锚点；**无 mermaid、无可编辑态** | 只读渲染采用上游；fork 的 TipTap 编辑态保留 |
| 1.6 | 文件变更监听 | `api/workspace-files`（`changes` stream）+ `client/resources` | 默认装载，成熟 | 上游 `fs.watch`/Chokidar、目录仅直系子项、文件级跟随；fork 为 `host.watchPath` + 手写重列 | 采用 `changes` stream（版本号 + absent 语义更完整） |
| 2.1 | Workspace 级人类终端 Tab 池（`ui-terminal`，`host.terminal*` + SSE） | `api/terminal-controller` + `client/ui-sidebar-terminal` | 默认装载，成熟 | 上游按 **Session** 隔离（`maxTerminals=8`、`maxAgentTerminals=4`），Tab=终端；传输是 Gateway mux 上的 **duplex `attach` 流**（带 seq/ack/writable），不是 SSE；fork 按 **Workspace** 池化 + SSE 帧 | 保留 fork 的 Workspace 池与 SSE；移植上游的 `retain` 窗口持有、无人值守回收、接管语义 |
| 2.2 | PTY 归属 | 上游：`ctx.subprocess.spawnTerminal`（`subprocess/subprocess-local`，底层 node-pty） | 成熟 | fork：`packages/host/apiproxy/src/human-terminal.ts` 直接 `node-pty` | 上游分层更干净（PTY 归 subprocess 能力面） |
| 2.3 | Agent 侧终端工具 | `packages/experimental/tool-terminal`（`terminal_*` 6 个工具）+ `experimental/terminal-panel` + `experimental/terminal-bundle` | 实验、默认关闭 | 上游把 Agent 终端做成「人能看见并接管的侧栏 tab」，人类首次键入即移交控制 | 接管语义值得移植；fork 现状是 Agent/人类两套互不相通 |
| 3.1 | Host 有头 Playwright + Tab 池 + screencast（`ui-browser`） | `client/ui-sidebar-browser` | Desktop 默认、Web 需 profile patch 打开 | 上游是**沙箱化 web 载体**：Web 用 iframe（可临时去沙箱）、Desktop 用 Electron `<webview>`；**Host 端无 Chromium**；不与 Agent 共享实例；应用自管历史 | 能力上 fork 更强（真 Chromium、下载/弹窗/登录态）；建议保留，但吸收上游「per-tab sandbox + 显式外部打开」的降级路径 |
| 3.2 | 人类/Agent 共享浏览器 | `browser-use/browser-use` seam + `experimental/browser-use-{runtime,playwright-mcp,chrome-devtools-mcp,stagehand-native}` | 实验、显式挂载 | 上游 Agent 浏览器 = MCP 驱动 Chromium，`headless` 默认 true；`attach` 模式独占外部 CDP 端点，**与人类侧栏浏览器无关** | fork 的「human 与 agent 同 Workspace Context 共享 tab id」是独有价值，保留 |
| 4.1 | Mermaid 渲染 + lightbox | **无对应**（上游整仓无 mermaid 依赖） | 上游无 | 上游 `ui-primitives` 只有 Shiki/KaTeX；`ui-renderer` 是 slot/React 渲染装配，不是 Markdown 渲染器 | 保留 fork `MermaidBlock` + `mermaid-load.ts`，建议上推 |
| 4.2 | `ZoomPanLightbox` 全屏缩放平移 | 部分：`ui-primitives` 的 `ZoomControls`、`DOCUMENT_ZOOM`、documentpreview 图片/PDF 缩放 | 成熟（控件与缩放语义） | 上游无「全屏 lightbox + 缩放平移 + reset」组合；图片 lightbox 不缩放 | 保留 fork lightbox，缩放范围语义对齐 `DOCUMENT_ZOOM`（0.25–4，step 0.25） |
| 5 | LSP 编辑器诊断（`lsp-editor` + `lsp-stdio` 诊断） | `lsp/lsp`、`lsp/lsp-stdio`、`lsp/tool-lsp` | Agent 导航成熟；诊断无 | 上游 README 明确「deliberately excludes ... rename, formatting, **diagnostics**, and symbol lists」，且每次查询重读文件、不保留文档状态 | 保留 fork 的 `lsp-editor` seam；它是上游 seam 的自然扩展，不冲突 |
| 6 | 工具箱 DetailsPanel 五段 Tab（`conversation.details.{tool,editor,git,terminal,browser}`） | `client/ui-sidebar-right` 的 `ctx.sidebarRightTabs` + `sidebar.right.pane.tab` | 默认装载，成熟（11+ 树外注册者） | 上游是**注册制**：任意包 `register({id,kind,patterns,priority,canOpen,...})` + keyed slot 即可加页/加 tab；fork 是 `ui-conversation` 静态声明的 5 个 `single` slot，树外插件无法加第 6 段 | **建议采用**：这是 pure win 的架构差异 |
| 7（邻近） | Git 面板 `ui-git`（工作区变更、stage/discard、提交/推送、图） | **无对应**（上游 `packages/client` 无 git UI；只有 Host 侧 agent 工具） | 上游无 | — | 保留 fork |

## 二、逐项详述

### 1. 文件浏览与编辑

上游相关包与关键路径（相对上游根）：

- `packages/client/ui-sidebar-files/README.md` — §What it registers / §The tree / §Search / §Reusable tree Factory。
  - tab 类型：`files`（`band: builtin`，无 patterns，走 `openTab('files')`）；正文注册在 keyed `sidebar.right.pane.tab`。
  - 工厂：`sidebar.files.navigator`（Session scope）、`sidebar.files.navigator.toggle`、`sidebar.files.tree`（root scope，带本地 `accessory` slot）。
  - 懒加载：每级展开时 `remote.workspaceFiles.list(sessionId, absolutePath)`；目录 watch 只跟直系条目；Search 走 Host 的 ripgrep（`rg --files`）。
  - 明确限制：「**Listing only.** No content search, artifact filter, drag-and-drop, or rename.」「One root ... no way to browse above it」。
- `packages/client/ui-sidebar-documentpreview/README.md` — §What it registers / §How it reads。
  - tab 类型 `text`（`band: fallback`，pattern `dsh-resource://file/**`，只收 Session 地址）。
  - 扩展点：`ctx.documentPreviews.register({ id, extensions, binaryExtensions?, priority, title, loading, wrap?, sourceLines? })` + keyed slot `sidebar.right.tab.document`。
  - 渲染器：Markdown、code、image（PNG/JPEG/GIF/WebP/BMP/ICO/SVG）、PDF、Office（本地转 PDF）、Excel、HTML（两种安全策略）、纯文本回退。
  - 读取：文本分页 `remote.workspaceFiles.read(...)`（滚动到底继续下一页），二进制 `readBytes`；`sidebars.right.tab.document.actions` / `.unpreviewable` 两个 list child slot。
  - 缩放：`ZoomControls`，范围 25%–400%，预设 Fit width/25/50/100/150/200。
- `packages/client/ui-sidebar-right/README.md` — §Extension seats / §ctx.sidebarRight / §State。多 Tab、拆分、浮动、布局持久化（`dsh.sidebar-right.v1.<sessionId>`）。
- `packages/client/ui-file-actions/README.md` — 保存/下载/复制图片；Desktop 才有 Save as；不写工作区。
- `packages/client/ui-file-drag/README.md` — 原图拖入 composer 或外部应用。
- `packages/api/workspace-files/README.md` — §Use this package / §The change feed / §Configuration。
  - 方法：`stat`、`read`（按行分页，`version`/`eof`）、`readBytes`（0 基字节窗，`Uint8Array`）、`list`、`search`、`changes`（stream，先 `ready` 再 `change`）。
  - 「**The service exposes no mutation operation.**」→ 上游 Host 没有人类写文件通道。
  - 配置：`maxBytes` 2 MiB、`maxFileBytes` 32 MiB、`maxLines` 5000、`maxEntries` 2000、`search.*`。

grep 结论：`monaco`/`CodeMirror` 在 `packages/` 下 0 命中（仅 `packages/experimental/inspector` 的 devtools 文档），代码预览用 Shiki 只读；`writeFile` 在 `packages/api`、`packages/client` 只出现在测试里。

→ fork `ui-file-editor` 的定位：上游**没有**编辑器。可对应的只有「树 + 只读多 Tab 预览 + 图片预览 + 目录监听」；「Monaco 编辑/保存、Markdown WYSIWYG、Add to Chat、新建/重命名/删除/移动」无上游对应物，属 fork 独有。

### 2. 人类终端

- `packages/api/terminal-controller/README.md` — §Use this package / §Agent terminals。
  - Remote `terminal`：`environment`、`shells`、`list`、`watch`、`create`、`retain`、`attach`、`rename`、`close`，全部按 Session 身份作用域。
  - PTY 归属：Host 用 `ctx.subprocess.spawnTerminal`（`TERM=xterm-256color`），**从不启动桌面终端**；底层 node-pty 在 `subprocess/subprocess-local`。
  - 传输：`attach` 是 duplex Remote stream（Gateway mux），带本地序号、`ack`、`writable`、`inputWindow`；新 attachment 恢复当前屏幕。
  - 恢复：headless xterm + serializer 生成开屏快照，`scrollback=1000` 有界；浏览器刷新可续，Host/进程重启不可续。
  - shell 发现：provider 默认 shell → `/bin/sh`/`cmd.exe` 回退 → `shellCandidates` 探测；`maxTerminals=8`、`maxAgentTerminals=4`、空闲回收 `unattendedTimeoutMs=7200000`。
- `packages/client/ui-sidebar-terminal/README.md` — shell 选择器、双击重命名、Take control、Reconnect、Agent/Sandbox 徽标、xterm 懒加载 chunk、按 Session 恢复（`dsh.terminal.binding.v1.*`）。
- `packages/terminal/terminal/README.md` — `ctx.terminals`：持久、owner-scoped（按 Agent 围栏）的终端会话服务，进程内、不跨重启。
- `packages/terminal/terminal-bash/README.md` — `shell` backend，POSIX bash / Windows pwsh，sandbox 策略下启动，行导向输出。
- `packages/experimental/tool-terminal/README.md` — 6 个模型可见工具：`terminal_open`、`terminal_send`、`terminal_read`、`terminal_signal`、`terminal_close`、`terminal_list`。
- `packages/experimental/terminal-panel/README.md` — 注册 `ctx.terminals` backend，使 `terminal_open` 变成人能看、能接管的侧栏 tab；人类首次键入即移交，之后 Agent 只读。
- `packages/experimental/terminal-bundle/README.md` — 把上述打包成 bundle，**Web/Desktop 默认关闭**。

→ fork `ui-terminal`（Workspace 级 tab 池 + `host.terminal{Profiles,Spawn,Write,Resize,Kill,List}` + `host.terminalStream` SSE + 断线回放）：上游的 Session 级终端池、duplex attach、retain/reclaim、Agent 接管语义都更强，但**没有** Workspace 级池，也没有 SSE 传输。

### 3. 人类浏览器

- `packages/client/ui-sidebar-browser/README.md` — §Use this package / §Understand the implementation / §Known Limitations。
  - Web：`sandbox="allow-scripts allow-forms allow-same-origin allow-popups allow-popups-to-escape-sandbox"` 的 iframe，应用自管历史；可临时去沙箱（不持久）。
  - Desktop：Electron `<webview>`，`keepMounted`，主进程审批 guest lease、导航与权限策略。
  - 默认状态：Web profile 通过 patch `disabled: false` 才启用；Desktop 默认启用（`bundle/web-app/cordis.patch.yml`：`disabled: !!js "ctx.get('profileContext')?.name !== 'desktop'"`）。
  - 协议策略：只收 HTTP(S)（含 loopback），拒绝 `file:`、data、blob、应用自身 origin。
- `packages/api/workspace-controller/README.md` — Workspace 增删改排 + `follow()` 投影；与浏览器载体无关（不是浏览器控制器）。
- `packages/browser-use/browser-use/README.md` — 一次只允许一个 browser-use provider。
- `packages/experimental/browser-use-playwright-mcp/README.md`、`browser-use-chrome-devtools-mcp/README.md`、`browser-use-stagehand-native/README.md`、`browser-use-runtime/README.md`、`browser-use-runtime/src/mcp.ts` — `mode: 'launch' | 'attach'`；`headless` **默认 true**；`attach` 用 CDP 端点且**独占**一个 Session；工具发现先于首个模型请求。

→ 上游人类浏览器**不是** Host 有头 Chromium，也**不与 Agent 共享实例**。fork `ui-browser`（Host Playwright 有头窗口、Workspace tab 池、screencast SSE、桌面 BrowserView occupant）是另一条路线：人类与 Agent 共享同一 Playwright Context/tab id（见 fork `packages/browser/tool-browser/README.md`：「Tools share tab ids with the toolbox 浏览器 segment」）。

### 4. Markdown / 预览增强

- `packages/client/ui-primitives/README.md` — 组件目录：`MarkdownText`（GFM + KaTeX + Shiki 流式增量）、`CodeBlock`、`ZoomControls`/`DOCUMENT_ZOOM`、`DraggableImage`、`ImagePreview`、`ImageLightbox`、`useCodeHighlighter`/`CODE_HIGHLIGHT_EXTENSIONS`/`languageForPath`。
- `packages/client/ui-primitives/src/document-zoom.ts` — `DOCUMENT_ZOOM = { minimum: 0.25, maximum: 4, step: 0.25, presets: [0.25,0.5,1,1.5,2] }`。
- `packages/client/ui-primitives/src/ImageLightbox.tsx` — 仅原图放大 + 关闭/Escape/焦点恢复，**无缩放平移**。
- `packages/client/ui-renderer/README.md` — 这是 slot/React 渲染装配层（`ctx.uiRenderer.mount`），**不是** Markdown 渲染器，无 mermaid/zoom 概念。

→ 结论：代码高亮（Shiki）与文档缩放（`ZoomControls`）上游已有且成熟；**mermaid 完全没有**；「全屏缩放平移 lightbox」上游没有（图片 lightbox 不支持缩放）。fork 的 `MermaidBlock` + `ZoomPanLightbox`（25%–400%、reset、close，被 mermaid 与图片共用）仍有价值。

### 5. LSP / 编辑器诊断

- `packages/lsp/lsp/README.md` — 只读导航：definition / references / implementations / hover；「Navigation is read-only and **deliberately excludes** generic JSON-RPC access, rename, formatting, **diagnostics**, and symbol lists」。
- `packages/lsp/lsp-stdio/README.md` — 按扩展名映射语言、每 workspace 按需起一个 server、**每次查询重读文件、查询间不保留文档状态**。
- `packages/lsp/tool-lsp/README.md` — 模型侧单个只读 `lsp` 工具，1-based UTF-16 位置。
- 目录核对：上游 `packages/lsp/` 只有 `lsp`、`lsp-stdio`、`tool-lsp`，**没有** `lsp-editor`；`packages/client` 无 diagnostic 相关编辑器 UI。

→ fork 的 `packages/lsp/lsp-editor`（`ctx.lspEditor`：provider 注册 + 每文件持久文档同步 + 规范化诊断给 Monaco markers + hover）与 `lsp-stdio/src/{editor-instance,diagnostics}.ts`（`publishDiagnostics` 版本化、settle 定时器、waiters）填补的是上游**刻意排除**的能力面。两者不冲突：上游 `lsp` 是 Service Definition，fork 的 `lsp-editor` 是并列 seam。

### 6. 侧边栏扩展点

上游（推荐参照）：

- `packages/client/ui-sidebar-right/src/client/contract/slots.ts` — 声明 6 个 seat：`rightbar.session`、`sidebar.right.pane.tab`（keyed，按 tab type `id` 派发）、`sidebar.right.pane.tab.title`（keyed）、`sidebar.right.tab.guide`（chain）、`sidebar.right.tab.guide.actions`、`sidebar.right.tab.guide.entry`（keyed）、`sidebar.right.tab.menu.item`（list）。
- `packages/client/ui-sidebar-right/README.md` §Extension seats — 两段式注册：静态定义 `ctx.sidebarRightTabs.register({ id, kind, patterns?, priority?, canOpen?, title, guide?, keepMounted? })` + keyed slot `sidebar.right.pane.tab` body；band 顺序 `extension` > `builtin` > `fallback`；同 `id` 二次注册抛错；`id` 是 seat key，所以树外包可以整体接管某个 kind。
- `docs/subsystems/sidebar-right.md` §Tab-type registration / §Shipped types / §Not built — 出厂类型：`guide`、`text`、`files`、`browser`、`subagentchat`。
- 树外注册证据（grep `sidebarRightTabs.register`）：`experimental/client-ui-forge`、`experimental/timeline`、`experimental/liveshare`、`experimental/session-inspector`、`client/ui-plan`、`client/ui-schedule`、`client/ui-subagent` 以及出厂 4 个类型 —— 共 11+ 处，证明「不改编排包即可新增 tab/页」。

fork（对照）：

- `packages/client/ui-conversation/src/client/contract/slots.ts:125-161` 静态声明 5 个 `single` slot：`conversation.details.{tool,editor,git,terminal,browser}`；`apply.ts:535-539` 同步声明；`client/skeleton/DetailsPanel.tsx:99-134` 硬编码 5 段渲染。
- 结论：树外插件**无法**新增第 6 段 tab；新增必须改 `ui-conversation` 的 contract + DetailsPanel + 各 occupant 的 owner props 类型。上游的 `sidebarRightTabs` 注册制是更优形态。

## 三、上游缺失（fork 独有价值）

1. **可写文件编辑器**：Monaco 0.52 多 Tab、显式保存、脏标记、脏 tab 守卫、Monaco↔磁盘 buffer 同步、`Add to Chat`（`MonacoSourceSelectionToolbar.tsx`、`composer-file-context-insert.ts`、`file-context-ref.ts`）。上游 Host 全部文件通道只读。
2. **Markdown WYSIWYG**：TipTap 3.30（`@tiptap/markdown` 往返），带只读 fenced block（mermaid/代码）内嵌；上游只有只读 Markdown。
3. **Mermaid**：`ui-primitives/src/markdown/{MermaidBlock.tsx,mermaid-load.ts}`，含 lightbox 与主题重绘；上游无 mermaid 依赖。
4. **ZoomPanLightbox**：全屏、缩放/平移/重置、图片与 mermaid 共用；上游 `ImageLightbox` 不缩放。
5. **Host 有头 Chromium + 人机共享**：Playwright 有头窗口（`packages/host/apiproxy/src/browser-registry.ts`，`headless` 默认 `false`）、screencast SSE、桌面 BrowserView occupant、人/Agent 共享 tab id（`packages/browser/tool-browser`）。上游人类浏览器是 sandboxed iframe/webview，与 Agent 完全隔离。
6. **Workspace 级人类终端池 + SSE**：`host.terminal*` RPC（`human-terminal.ts` 直接 node-pty、512 KiB scrollback 上限、Host 侧标题推导）；上游按 Session、走 duplex attach。
7. **编辑器 LSP 诊断**：`lsp-editor` seam + `lsp-stdio` 持久文档同步 + `publishDiagnostics` 版本化；上游显式排除。
8. **Git 面板**：工作区变更/暂存/丢弃/提交/推送/提交图，含与编辑器脏 tab 的守卫联动（`packages/client/ui-git`）；上游无任何 Git UI。

## 四、结论

1. 上游在「**只读浏览**」上已完整：文件树（懒加载 + rg 搜索 + 目录 watch）、文档预览（Markdown/代码/图片/PDF/Office/Excel/HTML）、图片缩放、Shiki 高亮、多 Tab/拆分/浮动、布局持久化。fork 对应部分若重写，应直接复用，不必自建。
2. 上游在「**人类写入**」上为零（`workspace-files` 无 mutation，全仓无编辑器内核），fork 的 `ui-file-editor`（Monaco + WYSIWYG + Add to Chat + 文件增删移）是真正不可替代的差异，应作为长期资产维护。
3. 终端两套实现各有强项：上游强在生命周期（Session 作用域、retain、空闲回收、Agent 接管、duplex ack 流），fork 强在作用域形态（Workspace 池）与 SSE 简单性。建议保留 fork 传输，移植上游的 retain/回收/接管语义。
4. 浏览器是根本性路线差异：上游人类浏览器是 sandboxed 载体（不与 Agent 共享、Web 端默认关）；fork 是 Host 真 Chromium 且人机共享。fork 能力更强但暴露面更大，建议补上游的 per-tab 沙箱降级与「外部浏览器打开」路径，而非回退。
5. Mermaid 与「全屏缩放平移 lightbox」上游完全缺失，fork 的 `MermaidBlock` + `ZoomPanLightbox` 保持必要；缩放范围语义建议对齐上游 `DOCUMENT_ZOOM`（0.25–4，step 0.25），以便未来共用。
6. LSP 诊断属上游**刻意排除**的能力面，fork 的 `lsp-editor` 不构成重复实现，而是对 `ctx.lspEditor` 的并行扩展；建议以「新增 seam」而非「改写 `lsp`」的方式长期维护。
7. 最大的架构欠账不在功能而在扩展点：fork 的 DetailsPanel 用 5 个静态 `single` slot 写死，上游 `ctx.sidebarRightTabs` 注册制已被 11+ 个树外包证明可用。若要让 fork 的编辑器/终端/浏览器/Git 成为「可插拔段」，应把 `conversation.details.*` 改造为「注册制 tab（kind + keyed body slot + 能力声明）」。
8. 优先级建议：① 采用上游 tab 注册制（结构性收益最高）；② 复用 `api/workspace-files` 的 `changes`/分页读取与 `documentpreview` 的图片/Office/PDF 渲染；③ 移植终端 retain/回收/接管；④ 保留 Monaco 编辑器、WYSIWYG、Mermaid、host 浏览器、Git、LSP 诊断作为 fork 独有能力。
