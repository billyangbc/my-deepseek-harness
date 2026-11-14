# S3：桌面壳 V5 与上游 0.2.2 桌面应用的取舍分析

- 分析对象 A（本 fork）：`<fork 工作区>`，`custom/main` HEAD `c5ac2488d9`；分叉点 `47f943859b`（2026-08-13，版本线 0.1.0-rc.x）。
- 分析对象 B（上游）：`<upstream 工作区>`，HEAD `39163dd30f`（2026-11-13，`0.2.2-rc.3`）。
- 分叉幅度：`git rev-list --count 47f943859b..39163dd30f` = **14926** 提交；fork 自身 `47f943859b..HEAD` = 214 提交。跨库 diff 需 `GIT_ALTERNATE_OBJECT_DIRECTORIES=<upstream>/.git/objects`。
- 本文只读代码，未安装依赖、未构建、未运行测试（两个工作区都没有可用的 `node_modules`）。

---

## 一、自制桌面壳的构成与独有行为清单

### 1.1 新增物

| 位置 | 规模 | 性质 |
|---|---|---|
| `apps/desktop` | `src` 27 个 `.ts`（2224 行）+ `tests` 21 个 spec + `electron-builder.yml` + 3 个 `scripts/`（bundle-shell / bundle-preload / brand-dev-electron） | Electron 壳本体 |
| `apps/desktop-host-pkg` | 仅 `package.json`（38 行，纯依赖清单） | 打包用 `pnpm deploy` 的依赖根，用于生成 `host-runtime` 闭包 |
| `packages/bundle/desktop-app` | `cordis.patch.yml`（实质）+ 空 `src/index.ts` | `desktop` profile 的组合层 |
| `scripts/{dev-desktop,prepare-desktop-packaging,run-electron-builder.mjs,smoke-desktop-packaging}.ts` | 284 行 | 开发 / 打包 / 冒烟 |
| `.github/workflows/desktop-packaging.yml` | CI 产出未签名 dmg/exe | 打包验证 |

### 1.2 对官方包的改动（桌面相关部分）

- `packages/boot/app-boot/src/profile.ts`：`PROFILE_TEMPLATES` 增加 `desktop: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-desktop-app']`。
- `apps/cli/src/args.ts`：新增 `dsh desktop` 子命令（`DesktopInvocation`、`allowUnknownOption`）；`apps/cli/src/desktop-launcher.ts` `spawnDesktop()` / `resolveElectronExecutable()` / `resolveDesktopMainEntry()`。
- `apps/cli/src/profile-boot.ts`：`INSTALL_ANCHOR` 与 `SHIPPED_PRESET_ROOT` 增加「上一级找不到就上两级」的打包布局回退。
- `packages/host/apiproxy`：新增 `browser-delivery.ts`（桌面 surface seam）、`browser-registry.ts`（Workspace 级 Playwright 池 + `browserDelivery: 'web' | 'desktop'`）、`browser-page-scripts.ts`、`browser-navigation-url.ts`，并新增 `host.browser.*` 系列 RPC。
- `packages/client/connection`：新增 `src/ipc-channels.ts`、`src/client/ipc-bridge.ts`（`DesktopIpcBridge` / `readDesktopIpcBridge`）、`src/client/ipc-api-client.ts`（`IpcApiClient extends AbstractApiClient`），`client/index.ts` 的 `apply()` 在 `readDesktopIpcBridge()` 命中时改用 IPC 客户端与 `createIpcConnectionRpc()`。
- `packages/client/modules/src/client/manifest.ts`：`window.__DSH_BOOT__` 之外的 `__DSH_HOST_BOOT__` 线格式；`packages/client/web/src/boot.tsx` 读取它；`AppRoot.tsx` 暴露 `retryHostBoot`。
- `apps/web/vite.desktop-dev.config.ts`：dev 时按 `.sessions/desktop-boot-graph.json` 注入 boot manifest 并从磁盘直接返回 `/plugins/<id>/client.js`。
- `packages/client/ui-browser`（新增包）：`src/client/browser-desktop-occupant.ts` 上报 occupant bounds。
- `packages/browser/tool-browser`（新增包）：`browser_*` 模型工具，经 `host.browser.*` RPC 与人类视图共用同一 Playwright 实例。

### 1.3 独有行为清单（自制的实质）

| # | 行为 | 关键文件 / 符号 |
|---|---|---|
| B1 | **Host 在 Electron Main 进程内 boot**（无子进程、无 HTTP 监听） | `apps/desktop/src/host-boot.ts` `DesktopHostController.boot()` → `runProfile({ profile: 'desktop', patchFiles: [], args: [] })`；`teardown()` → `shutdown.shutdown(0)` |
| B2 | **`desktop` profile 是一个不挂 `dsh-host-webserver` 的独立组合层** | `packages/bundle/desktop-app/cordis.patch.yml`（注释明写「without `dsh-host-webserver`, HTTP static serving, or loopback-only transport rows」）；`api-gateway` 行 `config.browserDelivery: desktop` |
| B3 | **IPC carrier 取代 HTTP/WS**：unary 走 `invoke`，下行 mux/host/watchPath/terminalStream/browserWatchScreencast 走 `send` 推送 | `apps/desktop/src/ipc-api-bridge.ts` `registerIpcApiBridge()`（`createHostApiFetchHandler(ctx, connection)` + `ctx.get('apiProxy')`）、`ipc-downlink.ts` `openApiProxyDownlink()`、`ipc-contract.ts` 通道常量；客户端 `packages/client/connection/src/client/ipc-api-client.ts` |
| B4 | **窄 preload 桥**：`contextBridge` 暴露 `window.dsh`，`delivery: 'desktop'` | `apps/desktop/src/preload.ts`（`fetch` / `cancelFetch` / `openStream` / `closeStream` / `onStream*` / `reportBrowserOccupantBounds` / `onRevealToolboxBrowser` / `openExternalUrl`） |
| B5 | **`dsh://app/` 自定义协议托管 SPA + 插件 client bundle** | `apps/desktop/src/protocol-dsh.ts` `DSH_PROTOCOL_SCHEME='dsh'`、`DSH_APP_AUTHORITY='app'`、`resolveDshProtocolPath()`（`/plugins/<id>/client.js`、`/monaco/*`）；`main.ts:74` `registerSchemesAsPrivileged` |
| B6 | **Host 启动时扫描 Loader 条目现场合成 boot graph** | `boot-graph.ts` `composeDesktopBootGraph()`（`ctx.loader.entries()` → `dsh.client.platform === 'web'` → `exports['./client']` → sha1 12 位 rev）；`spa-index.ts` `buildDesktopSpaIndexHtml()` |
| B7 | **有头 Chromium 与人类视图共用一个渲染面**：Main 持 `BrowserView`，Host 用 `playwright.chromium.connectOverCDP(127.0.0.1:9222)` 附着 | `browser-view-manager.ts` `DesktopBrowserViewManager`（`ensureTab` / `pageForTab` / `selectTab` / `closeTab` / `cdpEndpoint()`）；`main.ts:67-72` `remote-debugging-port` + `remote-allow-origins=*`；`browser-delivery.ts` `setDesktopBrowserSurface()` / `requireDesktopBrowserSurface()` |
| B8 | **occupant bounds**：Renderer 上报工具箱浏览器段的屏幕矩形，Main attach/detach BrowserView | `browser-view-bounds.ts` `applyBrowserOccupantBounds()`；`browser-bounds-ipc.ts` `registerBrowserBoundsIpc()`；`main.ts:135-141` `wireDesktopBrowserSurface()`；客户端 `browser-desktop-occupant.ts` `readDesktopBrowserOccupantReporter()` |
| B9 | **退出守卫（渲染层未保存编辑器）**：Main 发 `dsh:exit-request`，Renderer 回 `proceed` 才 teardown Host | `exit-guard.ts` `createExitGuardCoordinator()`；`main.ts:89-99 / 199-209 / 313-315` |
| B10 | **窗口几何持久化** | `window-bounds.ts` `loadWindowBounds()` / `saveWindowBounds()` / `WINDOW_BOUNDS_STORAGE_KEY='desktop.windowBounds.v1'`；`main.ts:237` 应用、`204-207` 保存 |
| B11 | **单实例 + 二次启动聚焦** | `single-instance.ts` `installSingleInstanceLock()`；`main.ts:281-288` |
| B12 | **应用菜单（About / Settings / Quit + 仅剪贴板角色的 Edit）** | `app-menu.ts` `buildApplicationMenuTemplate()`（注释解释为何不用 `role: 'editMenu'`：会绕过 Monaco/TipTap 的 undo） |
| B13 | **打包捆绑 Playwright Chromium**，并设 `PLAYWRIGHT_BROWSERS_PATH` | `packaging-paths.ts` `DESKTOP_EXTRA_RESOURCE_NAMES`（`web-dist` / `playwright-browsers` / `host-runtime`）；`packaging-env.ts` `applyPackagedRuntimeEnv()`；`scripts/prepare-desktop-packaging.ts`（`pnpm deploy --legacy --prod --node-linker=hoisted` + `playwright install chromium`） |
| B14 | **品牌 NanGeAGI** | `app-branding.ts` `DESKTOP_APP_DISPLAY_NAME='NanGeAGI'`；`electron-builder.yml` `productName: NanGeAGI`、`appId: ai.deepseek.harness.desktop` |
| B15 | **attach 模式**：连外部 `dsh web`，只退出 GUI、不 boot Host | `attach.ts` `readDesktopAttachConfig()` / `shouldSkipHostBoot()`；`load-url.ts` `resolveDesktopLoadTarget()` |
| B16 | **窗口打开策略**：http(s) 弹窗改投工具箱浏览器，其余丢弃 | `window-open-policy.ts` `decideDesktopWindowOpen()`；`main.ts:271-279` |
| B17 | dev 工作流：`dsh desktop` / `pnpm run dev:desktop` / Vite 5173 | `scripts/dev-desktop.ts`；`apps/web/vite.desktop-dev.config.ts` |
| B18 | 打包冒烟 + CI（未签名产物） | `apps/desktop/src/artifact-smoke.ts`；`scripts/smoke-desktop-packaging.ts`；`.github/workflows/desktop-packaging.yml` |

---

## 二、上游 0.2.2 桌面应用能力盘点

### 2.1 包与入口

| 包 / 目录 | 角色 |
|---|---|
| `apps/desktop`（`@deepseek-ai/dsh-desktop`，1619 行 `main.ts` + 约 11.2k 行 `src` + 约 200 个测试） | Electron 壳：窗口、协议、更新、托盘、命令安装、插件管理、欢迎页、多语言 |
| `apps/desktop-host`（`@deepseek-ai/dsh-desktop-host`） | 私有 Host 进程入口：`src/index.ts` `runProfile()`、`src/cli.ts` `runDesktopCli()` |
| `apps/desktop-publisher` | 发布链路（腾讯云 COS、灰度、安装包上传、回滚） |
| `packages/host/authentication-desktop` | 父进程 → 子进程的一次性凭据 provider |
| `packages/client/ui-sidebar-browser` | 右栏浏览器：Renderer `<webview>` + Main 租约 |
| `packages/browser-use/*`、`packages/computer-use/*` | Agent 侧浏览器自动化（MCP：chrome-devtools / playwright / stagehand）与 CUA |

### 2.2 Host 启动方式

- Main 用 Electron RunAsNode **子进程**启动 `@deepseek-ai/dsh-desktop-host`：`apps/desktop/src/host-process.ts`（`ReadyEvent` / `FatalEvent` / `PlatformSessionEvent` / `quit-inspection` / `update-tasks` / `file-drag-resource`，全部经 Node IPC 回传）。
- 子进程内 `apps/desktop-host/src/index.ts`：`receiveDesktopSecret(process)` 取父进程下发的凭据 → `loadProfileDirectory('dsh', projectDir, installAnchor)` → `runProfile({ profile: 'desktop', args: ['--no-open', '--host', '127.0.0.1', '--port', '0'] })` → 发 `ready`（含 Host URL 与 **启动注入表**）。
- 凭据模型：`packages/host/authentication-desktop` + `apps/desktop/src/parent-authentication.ts`，Electron 每次启动生成随机凭据，只在 Main 内存，渲染层拿不到（`apps/desktop/README.md` 有完整描述）。

### 2.3 Carrier

- **HTTP `/api` + WebSocket 下行**：`packages/client/connection/src/http-bridge.ts` `bridge()`、`websocket-downlink.ts`、`rpc.ts` `createSharedFetchHandler('/api')`。
- 渲染层加载 `dsh-app://app/`；`apps/desktop/src/main.ts` `protocol.handle(SCHEME, …)`：静态资源从 `dsh-web-frontend/dist` 直出，其余请求 `forwardWebRequest(request, hostUrl, hostCookie)` 转发给 Host（同源效果）；WS 升级请求在 `session.defaultSession.webRequest.onBeforeSendHeaders` 里补 `origin`/`cookie`。
- 启动注入：`ipcMain.handle(DESKTOP_IPC.boot)` 返回 `{ injections, streamBaseUrl }`；页面侧由 `packages/client/web/src/apply-injections.ts` `applyIndexInjections()` 执行（`IndexInjection` 来自 `@deepseek-ai/dsh-host-webserver`）。
- **传输可替换性已是公开 seam**：`ConnectionFetchHandler`（`packages/client/connection/src/rpc.ts:238`，注释写明「Transport-independent Fetch handler used by HTTP and worker carriers」）、`ClientConnectionRpc`（`rpc.ts:256`）；`packages/experimental/webworker-runtime/src/worker-host.ts:268` 就是一个**非 HTTP 载体**的现成先例（`connection.createSharedFetchHandler('/api')` + `connection.browserPeer`）。

### 2.4 浏览器整合

- 人类视图：Renderer `<webview>`（`apps/desktop/src/main.ts` `webviewTag: primary`）+ `packages/client/ui-sidebar-browser/src/client/electron/ElectronWebViewImpl.ts` / `ElectronWebviewPresentation.ts`；Main 侧 `apps/desktop/src/browser-guests.ts` `DesktopBrowserGuests.acquire/release/bind` 发放 `DesktopBrowserLeaseId`、按 workspace 分配 `session.fromPartition(...)`，并在 `will-attach-webview` 里强制 `sandbox/contextIsolation/nodeIntegration:false` 等一整套硬化。
- 账号页：`apps/desktop/src/platform-view.ts` `DesktopPlatformView`（`WebContentsView` + 账户分区 + cookie 清理）。
- Agent 视图：`packages/browser-use/browser-use`（唯一 provider 注册位）+ MCP 实现；**没有** CDP 附着、**没有** screencast RPC（全库 `connectOverCDP` / `chromium.launch` / `screencast` 均为 0 命中）。

### 2.5 打包与运行时闭包

- `apps/desktop/scripts/electron-builder-config.mjs`（`createElectronBuilderConfig()`）：`asar: true`、`asarUnpack` 含 `.node/.dylib/.dll`、LibreOffice kit、CUA 原生包；`protocols: [{ schemes: ['dsh'] }]`（OS 深链）；`productName: 'DeepSeek Harness'`；`appId` 由 `DSH_DESKTOP_APP_ID` 经 `resolveDesktopAppId()` 校验；`electronFuses.runAsNode: true`。
- 签名/公证/安装器/更新：`macos-signing-keychain.mjs`、`notarize-macos*.mjs`、`windows-sign.mjs`（token 签名）、`installer/*.nsh` + `window-frame.cpp`（自绘 NSIS 安装器 + 8 种语言）、`installed-update-*`、`updater/electron-updater/*`。
- 运行时闭包：`apps/desktop/src/core-package-set.ts`（`DESKTOP_PACKAGE_SET_FILE` / `DESKTOP_HOST_PACKAGE='@deepseek-ai/dsh-desktop-host'`，dsh 与 desktop-host 的**签名 npm tarball 集合**）、`runtime-tree.ts`、`scripts/prepare-package-set.ts`、`prepare-primary-runtime.ts`（内置 pnpm + Node）。
- 卸载清理：`docs/user/guide/desktop-uninstall-cleanup.md`（逐条列出 `~/.dsh`、`Application Support/@deepseek-ai/dsh-desktop`、`Preferences/com.deepseek.dsh*`、Caches、`ShipIt` 等）、`apps/desktop/src/owned-directory.ts` `removeOwnedDirectory()`（不跟随符号链接/junction）、`update-cleanup-log.ts`、`installer/uninstall*.nsh`。

### 2.6 profile

- 桌面 profile **不是**独立 bundle：`apps/desktop/src/project-manager.ts` `WEB_PROFILE = PROFILE_TEMPLATES.web`，`createRuntimeProjectMetadata()` / `createDevelopmentProjectMetadata()` 写的都是 `dsh.profile.bundles = ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-web-app']`。
- 桌面专属行为由 **官方 `web-app` bundle 内的条件行**提供：`packages/bundle/web-app/cordis.patch.yml` 里 `disabled: !!js "ctx.get('profileContext')?.name !== 'desktop'"`（`desktop-product-telemetry`、`product-analytics`、`ui-sidebar-browser`、`workspace-dependencies`、`skill-office` 的 `pythonEnvironment`）。
- 名字被保留：`apps/cli/src/args.ts:84` —— `if (profile.toLowerCase() === 'desktop') program.error('error: profile "desktop" is managed exclusively by the Electron application')`；profile 目录由 `apps/desktop/src/paths.ts` 固定为 `$DSH_HOME/profiles/desktop`。
- `packages/util/official-bundles/src/bundles.ts` 的 `PROFILE_TEMPLATES` 只有 `acp / web / headless / sdk / sdk-minimal`，**没有 desktop**。

### 2.7 其它原生能力

单实例（`single-instance.ts` `claimDesktopSingleInstance`）、退出确认（`quit-confirmation.ts` `DesktopQuitConfirmation` + Host 侧 `quit-inspection`，`QUIT_INSPECTION_DEADLINE_MS = 2000`）、Windows 托盘（`tray.ts`）、快捷键与本地化菜单（`keyboard.ts`、`locale.ts` 1606 行、`tests/expected/application-menu-*.json`）、目录选择/麦克风权限/文件拖拽/`dsh` 命令安装与卸载（`command-installation.ts`、`command-management.ts`）、自带 pnpm 的插件管理（`apps/desktop-host/src/cli.ts` `manageDesktopProfile: true`）、崩溃报告与致命恢复（`crash-report.ts`、`fatal-recovery.ts`）、产品埋点、强制更新与灰度（`mandatory-update-*`、`packages/host/product-telemetry-otel`）。

---

## 三、逐项对比表

图例：**删** = 上游已原生具备，删自制；**移** = 上游缺失，值得以独立插件/包保留；**弃** = 与上游冲突必须放弃；**改** = 保留意图但换个接法。

| 能力 | 自制实现（文件 / 符号） | 上游 0.2.2 实现（文件 / 符号） | 取舍 | 迁移代价 |
|---|---|---|---|---|
| Host 启动 | Main 进程内 `DesktopHostController.boot()`（`apps/desktop/src/host-boot.ts`） | 子进程 `host-process.ts` → `apps/desktop-host/src/index.ts` `runProfile()` | **弃**：进程内 boot 让任一 Host 插件异常直接带走整个壳，且拿不到上游的致命恢复/崩溃报告/更新前停 Host | L（等价改写，收益是稳定性） |
| RPC carrier | IPC `invoke` + `send`（`ipc-api-bridge.ts` / `ipc-api-client.ts`） | HTTP `/api` + WS 下行（`http-bridge.ts`、`websocket-downlink.ts`、`createSharedFetchHandler`） | **移**（可选）：作为独立「IPC carrier 插件对」，不替换官方默认；上游 seam 已存在（`ConnectionFetchHandler` / `ClientConnectionRpc`） | M（host 半 ~150 行 + client 半 ~300 行，重点是流式 `wireStream` 与 `Peer` 适配） |
| 认证 / 信任边界 | 无（同进程 IPC，无 token） | 父→子一次性凭据 + `owner` Peer + `dsh-desktop-session` Cookie；`assertDesktopSender()` 校验 IPC 来源 frame | **弃**自制（无认证模型）；**采用**上游 | — |
| SPA 托管 | `dsh://app/` 从 `apps/web/dist` 直出（`protocol-dsh.ts`） | `dsh-app://app/`：静态从 `dsh-web-frontend/dist`，其余 `forwardWebRequest()` 带 cookie 转发 | **删**：上游方案更完整（真同源、无需 boot graph 现场合成） | S（删代码） |
| 插件 client bundle 分发 | `boot-graph.ts` `composeDesktopBootGraph()` + `protocol-dsh` 的 `/plugins/<id>/client.js` | `dsh-client-modules` node 半 + `@deepseek-ai/dsh-host-webserver` 的注入表 / 静态路由 | **删**（功能重复） | S |
| 面板内浏览器（人类） | Main `BrowserView` + IPC occupant bounds（`browser-view-manager.ts`、`browser-view-bounds.ts`） | Renderer `<webview>` + Main 租约（`browser-guests.ts`、`ElectronWebViewImpl.ts`） | **改/移**：上游 `webview` 路径更安全且已被布局托管；但自制路径换来「Agent 与人类同一 webContents」 | M（若只要人类视图，直接用上游；若要共面见下一行） |
| **Agent 与人类共用同一浏览器** | `connectOverCDP` 附着 Electron（`browser-registry.ts` + `browser-view-manager.ts`）+ `host.browser.*` RPC + `tool-browser` | **无对应能力**（`connectOverCDP` 全库 0 命中；`browser-use` 走独立 MCP 浏览器，`ui-sidebar-browser` 只服务人类） | **移**：这是本 fork 最有价值的独有能力，做成独立包保留 | L（需把 `host.browser.*` 从旧 `apiProxy` 面重做上游 Typert Remote，并重建 occupant 通道） |
| 浏览器 occupant bounds | `applyBrowserOccupantBounds()` + `reportBrowserOccupantBounds` | 无（`webview` 由 CSS 布局托管，不需要） | 随上一行一并**移** | S（含在上面） |
| 单实例 | `installSingleInstanceLock()` | `claimDesktopSingleInstance()` | **删** | S |
| 退出守卫 | Renderer dirty-editor 往返（`exit-guard.ts`，无超时） | Host `quit-inspection` + `DesktopQuitConfirmation`（2s 超时，未知即确认） | **改**：语义是两件事（未保存编辑器 ≠ 有任务在跑），应在上游确认框之外保留一个 `beforeunload` 式守卫，但必须带超时 | S–M |
| 窗口几何持久化 | `window-bounds.ts`（`desktop.windowBounds.v1`） | **无**（`main.ts` 固定 1280×820，仅 fullscreen 用 `getBounds()`） | **移**：小而有用的上游缺口 | S（可做成 60 行独立模块或反向提 PR） |
| 应用菜单 / 托盘 / 快捷键 / 本地化 | `app-menu.ts`（英文硬编码三项 + 剪贴板角色） | `keyboard.ts` + `locale.ts`（8 语言，`tests/expected/application-menu-*.json` 快照）+ `tray.ts` | **删**自制，采用上游 | S |
| 深链 `dsh:` scheme | 自制把 `dsh:` 当内部协议 | `electron-builder.config.mjs` `protocols: [{ schemes: ['dsh'] }]`（OS 深链） | **弃**自制用法（同一 scheme 两种语义会互相踩） | S |
| 打包 / 签名 / 公证 / 安装器 / 更新 / 发布 | `electron-builder.yml`（`identity: null`、`hardenedRuntime: false`、`signAndEditExecutable: false`、默认 NSIS） | `scripts/electron-builder-config.mjs`、`windows-sign.mjs`、`notarize-macos*.mjs`、`installer/*`、`updater/*`、`apps/desktop-publisher/*` | **弃**自制，采用上游 | S（直接删；但想要更新能力就必须用上游链路） |
| 卸载残留治理 | **无**（无卸载器定制、无清理文档） | `docs/user/guide/desktop-uninstall-cleanup.md`、`owned-directory.ts`、`update-cleanup-log.ts`、`uninstall*.nsh` | **弃**自制，采用上游 | S |
| 外部插件 / 插件管理 | 无（`desktop` profile 固定，`host-runtime` 打包死） | `command-installation.ts`、`command-management.ts`、`apps/desktop-host/src/cli.ts`（`manageDesktopProfile`）+ 自带 pnpm | **弃**（自制这条路走不通，用的是 workspace 内固定包集） | — |
| Playwright Chromium 捆绑 | `extraResources` + `PLAYWRIGHT_BROWSERS_PATH` | **无**（桌面不带 Playwright；`browser-use` 用 MCP provider） | 随「共面浏览器」一起**移**（是那项能力的必要成本：约 +150–300MB） | M（要接上游 asarUnpack/签名流程） |
| profile 名称与桌面行为注入 | 新增 `PROFILE_TEMPLATES.desktop` + 独立 `dsh-desktop-app` bundle | profile 名 `desktop` 被 `args.ts:84` 保留；桌面行为由官方 `web-app` 的 `profileContext.name === 'desktop'` 条件行提供 | **弃**：不能再叫 `desktop`；建议 `nange-desktop`（见 §4） | M（要重写组合层与条件判定） |
| dev 工作流 | `scripts/dev-desktop.ts` + `vite.desktop-dev.config.ts` + `dsh desktop` | `apps/desktop/scripts/dev.ts`（准备 development project / primary runtime / 启动 Electron） | **删**自制，采用上游 | S |
| 打包冒烟 / CI | `artifact-smoke.ts`、`smoke-desktop-packaging.ts`、`desktop-packaging.yml` | `apps/desktop/tests/*.built.e2e.ts`、`installed-docs-asar.built.e2e.ts`、`packaging-run.mjs` 等 | **删** | S |
| 品牌 | `app-branding.ts` `NanGeAGI`、`appId: ai.deepseek.harness.desktop`、`copyright: Copyright © DeepSeek` | `productName: 'DeepSeek Harness'`、`resolveDesktopAppId(env)`、`resources/icon-*`、`packages/client/ui-brand-official` | **弃**（`ai.deepseek.*` appId 与官方安装包同域，且版权行与品牌名不匹配；改名必须连 appId 一起换） | S |

**净结论（三条）**

1. 上游 0.2.2 的桌面应用不是「缺一块」，而是**已经比自制壳完整一个数量级**：carrier、认证、SPA 托管、插件分发、签名公证、安装器、更新、卸载清理、多语言、托盘、插件管理、崩溃恢复全部原生。自制壳里 **12 项可直接删除**。
2. 自制壳真正独有的只有 **两项**：**（a）Agent 与人类共用同一有头 Chromium（BrowserView + CDP + occupant bounds + `host.browser.*` + `tool-browser`）**，**（b）窗口几何持久化**（外加一个可选的 IPC carrier 试验）。前者是 ADR-0007/0008/0010 的核心资产，后者是 60 行的小缺口。
3. 自制壳的**架构前提本身就与上游冲突**：`desktop` profile 名、`dsh:` scheme、「不启 webserver」这三条都被上游显式占位或结构性地依赖。继续沿这条线走，等于永久分叉 `apps/cli`、`packages/boot/app-boot`、`packages/client/connection`、`packages/host/apiproxy` 四个官方包。

---

## 四、若保留：如何做成「不改官方包」的独立交付物

### 4.1 包边界（建议三个独立包，均不修改 `packages/**` 官方包）

```
@nange/dsh-desktop-browser      ← 纯能力包（最高价值，优先做）
  src/host/    BrowserRegistry（Playwright 池）+ Typert Remote 端点 host.browser.*
  src/client/  BrowserPanel（工具箱浏览器段）+ occupant bounds 上报
  src/tool/    browser_* 工具（消费上面的 Remote，不直接 import playwright）
  src/electron/ BrowserView 宿主（只在 Electron Main 加载，动态 import）

@nange/dsh-desktop-window       ← 可选小包：窗口几何持久化 + 单实例 + occupant 通道
  （若最终采用上游桌面壳，这一包可以缩成「窗口几何」一条，甚至直接反向提 PR）

@nange/dsh-ipc-carrier          ← 可选试验包：IPC carrier 插件对（host 半 + client 半）
```

组合方式：**不改官方 profile 模板**，用用户层 `$DSH_HOME/profiles/<name>/cordis.patch.yml` 追加自己的 bundle；桌面壳则直接复用上游 `apps/desktop`，把上面三个包作为**树外插件**装进 `$DSH_HOME/profiles/desktop`（正是上游 `command-installation.ts` / `Plugin Manager` 支持的路径）。

### 4.2 上游已经具备、可直接消费的 seam

| seam | 位置 | 用法 |
|---|---|---|
| 传输无关 fetch handler | `packages/client/connection/src/rpc.ts` `ConnectionFetchHandler`；`HostConnectionService.createSharedFetchHandler('/api')` | IPC carrier 的 host 半只需 `handler.fetch(request, peer)` |
| 进程内 Peer | 同上 `ctx.connection.browserPeer`、`createPeer({ principal })` | 免认证地代表本机 owner 派发 |
| 客户端 RPC 抽象 | `packages/client/connection/src/rpc.ts:256` `ClientConnectionRpc`；`client/rpc.ts` `createWebConnectionRpc()` | IPC carrier 的 client 半实现同一接口，`apply()` 里按存在性选择 |
| 启动注入表 | `@deepseek-ai/dsh-host-webserver` 的 `IndexInjection` + `packages/client/web/src/apply-injections.ts` `applyIndexInjections()` | 独立壳注入 boot 数据不必自有 `window.__DSH_BOOT__` 语义 |
| Electron 宿主协议 | `apps/desktop/src/main.ts` `protocol.handle('dsh-app', …)` | 复用官方 `dsh-app://app`，不要自造 `dsh://` |
| 浏览器 guest 租约 | `packages/client/ui-sidebar-browser/src/types.ts` `DesktopBrowserBridge` / `DesktopBrowserLeaseId` | 若要人类视图继续用 `webview`，直接接这套 |
| 桌面条件行 | `packages/bundle/web-app/cordis.patch.yml` 的 `profileContext.name === 'desktop'` | 需要官方 bundle 内新增条件时，改自己的用户层 patch 用 `!!js` 覆盖**同名行 config**，而不是改官方 yml |

### 4.3 上游缺口（必须自持，无法回避）

1. **没有 `host.browser.*` RPC 面**：`packages/host/apiproxy`（fork）里的 40+ 个 browser RPC 在上游无对应物；上游的浏览器自动化是 `browser-use` 的 MCP provider，人类视图是 `webview`，两者不共享实例。→ 自有的 Remote 端点、schema、client 契约全部自己维护。
2. **没有 CDP 附着能力**：上游无 `connectOverCDP`，因此「Agent 操作人类看到的同一个 webContents」必须自带 registry，且必须自己解决多 webContents 下的 target 选择（fork 用 `findCdpPageByUrl()` + URL 归一化，这部分逻辑（`desktop-browser-cdp.ts`、`browser-page-scripts.ts`）价值高、可原样搬）。
3. **没有 screencast**：`browserWatchScreencast` 在上游 0 命中；如果保留「浏览器交付」（非桌面）形态，这条 SSE 通道也要自持。
4. **没有窗口几何持久化**：见 §3。
5. **无 webserver 的 profile 组合层官方不提供**：自制 `dsh-desktop-app` 的「去掉 webserver」在上游会成为持续分叉（`applyIndexInjections` 的类型来自 webserver 包、`connection` 的 node 半在没有 `webServer` 时短路）。→ **建议放弃「无 webserver」**，只保留上游默认的 `127.0.0.1` 随机端口；这样 §4.1 的三个包才能完全不碰官方包。
6. **`desktop` profile 名不可用**、`dsh:` scheme 被深链占用、`appId` 必须换。→ 命名方案：profile `nange-desktop`；scheme 复用官方 `dsh-app:`；appId 用独立反域名（如 `chat.nange.dsh.desktop`）。

### 4.4 落地顺序（按收益 / 成本比）

1. **先弃**：删 `apps/desktop/{src,scripts}`、`apps/desktop-host-pkg`、`packages/bundle/desktop-app`、`scripts/*desktop*`、`desktop-packaging.yml`、`apps/desktop/electron-builder.yml`；回滚 `apps/cli/src/args.ts`、`apps/cli/src/desktop-launcher.ts`、`packages/boot/app-boot/src/profile.ts`、`packages/client/web/src/boot.tsx`、`AppRoot.tsx`、`packages/client/modules/src/client/manifest.ts`、`apps/web/vite*.config.ts` 的壳相关改动。
2. **再移**：把 `browser-registry.ts` + `browser-page-scripts.ts` + `browser-navigation-url.ts` + `ui-browser` + `tool-browser` 从「改官方 apiproxy」重构成「自带 Typert Remote 端点」的树外包；这是唯一值得投入的部分。
3. **后补**：窗口几何 / 单实例 / occupant 三个小件按需保留；IPC carrier 先不做（上游 HTTP 方案已足够，且省掉认证模型）。
4. **回收测试**：`apps/desktop/tests/{browser-view-manager,browser-view-bounds,desktop-browser-cdp,browser-bounds-ipc}.spec.ts` 是纯逻辑、可平移；其余壳测试随壳一起删。

---

## 五、卸载残留与风险

### 5.1 当前自制的卸载残留（实测路径）

| 残留 | 来源 | 说明 |
|---|---|---|
| `~/Library/Application Support/NanGeAGI/`（Windows `%APPDATA%/NanGeAGI`） | `main.ts:54` `app.setName('NanGeAGI')` → Electron `userData` | 内含 `desktop.windowBounds.v1`（`window-bounds.ts`）；electron-builder 默认 NSIS 卸载**不删** |
| `~/.dsh/profiles/desktop/` | `host-boot.ts` → `runProfile({ profile: 'desktop' })` + `PROFILE_TEMPLATES.desktop` | Host boot 即写 profile，`package.json` 里 `dsh.profile.bundles = ['@deepseek-ai/dsh-base','@deepseek-ai/dsh-desktop-app']`；**同时抢占上游桌面应用的保留目录** |
| `~/.dsh/` 其余共享数据（sessions / credentials / settings / storages） | 与 CLI 共享 | 卸载桌面壳不该删；但用户无法区分，官方文档提供了可选清理脚本 |
| `dist/desktop/staging/`、`dist/desktop/installers/` | `scripts/prepare-desktop-packaging.ts`、`electron-builder.yml` | 开发机残留，`pnpm run clean` 未必覆盖 |
| Playwright 浏览器 | `extraResources/playwright-browsers` | 随 app 删除；但 `PLAYWRIGHT_BROWSERS_PATH` 只在 packaged 模式设置，dev 模式仍写用户缓存目录 |

对比上游：`docs/user/guide/desktop-uninstall-cleanup.md` 逐条列出 9 个 macOS 路径 / 3 个 Windows 路径，`owned-directory.ts` `removeOwnedDirectory()` 还专门处理「递归 rm 会跟随 Windows junction」的坑。自制壳在这一维度是**空白**。

### 5.2 风险清单

1. **安全：CDP 常开且放开来源**。`main.ts:69-72` 无条件 `remote-debugging-port=9222` + `remote-allow-origins=*`（只在 attach 模式关）。任何本机进程（含浏览器页面）都能连 `127.0.0.1:9222` 接管 Electron 全部 webContents，包括持有 `window.dsh` IPC 桥的主窗口 → 等价于拿到 Host 全量 RPC。建议：随机端口、`remote-allow-origins` 收窄到 `devtools://devtools`、仅 dev 开启。
2. **安全：IPC 无 sender 校验**。`main.ts:305-315` 的 `ipcMain.handle` 未校验 `event.senderFrame`；`attach.ts` 允许把**任意 http(s) URL** 载入主窗口，该页面同样能调用 `dsh:open-external-url`、`reportBrowserOccupantBounds` 等。上游 `apps/desktop/src/ipc.ts` `assertDesktopSender(event, ['app'])` 是必需的对照实现。
3. **安全：`dsh://` 权限过宽**。`registerSchemesAsPrivileged` 给了 `corsEnabled: true` + `supportFetchAPI: true`，而 `resolveDshProtocolPath()` 的根检查是字符串前缀比较（`startsWith(normalize(distRoot))`），`..` 之外的软链接未处理；上游的 `dsh-app://app` 只暴露固定路径白名单。
4. **生命周期：Host 与壳同生共死**。Host 在 Main 内 boot，任一 Host 插件抛未捕获异常会带走整个 Electron；上游用子进程 + `fatal-recovery.ts` + `crash-report.ts` 隔离。`retryHostBoot()` 只在首次 boot 失败时可用，Host 中途死掉无法恢复（要整进程重启）。
5. **生命周期：退出可能卡死**。`exit-guard.ts` 的 `requestQuit()` 返回的 Promise 只有在 Renderer 回 `dsh:exit-guard-result` 时兑现，**没有超时**；`mainWindow.on('close')` 又 `preventDefault()`，所以渲染进程崩溃/白屏时用户关不掉窗口（只能强杀）。上游 `QUIT_INSPECTION_DEADLINE_MS = 2000` 把超时当「有任务在跑」处理。
6. **打包：完全不签名**。`electron-builder.yml`：`mac.identity: null`、`hardenedRuntime: false`、`dmg.sign: false`、`win.signAndEditExecutable: false`。macOS 会被 Gatekeeper 隔离、无公证；Windows SmartScreen 全量告警；且没有证书就没有 electron-updater 的可信更新链路。
7. **品牌/合规**：`productName: NanGeAGI` + `appId: ai.deepseek.harness.desktop` + `copyright: Copyright © DeepSeek` 三者互相矛盾；`ai.deepseek.*` 命名空间属于上游，同一 appId 的官方安装包与本壳会互相覆盖安装/卸载/更新状态。改名必须**同时**换 appId，否则残留目录（§5.1）无法迁移。
8. **名称冲突**：`desktop` profile 与 `$DSH_HOME/profiles/desktop` 被上游桌面应用独占（`apps/cli/src/args.ts:84`）；一旦用户装过官方桌面，自制壳与官方会**写同一个 profile 目录**，bundle 列表互相覆盖（自制写 `dsh-desktop-app`，官方写 `dsh-base + dsh-web-app`）。
9. **维护成本**：自制基线是 0.1.0-rc.x（分叉点后上游 14926 个提交）。它依赖的 `AbstractApiClient`、`createHostApiFetchHandler`、`createHostApiFetchHandler(ctx, connection)`、`toFetchHandler`、旧 `apiProxy` 面在上游**已不存在**（上游改为 `dsh-api-gateway` + Typert + `api/remotes`），因此「把 IPC carrier 拿到上游」不是移植而是重写。
10. **Playwright 版本绑定**：`apps/desktop/package.json` 与 `apps/desktop-host-pkg/package.json` 都把 `playwright` 固定为 `^1.49.0`（上游已到 0.2.2 的工具链），Chromium 二进制与 Electron 35.7.5 的 CDP 协议版本需同时对齐；升级任一侧都会打破 `connectOverCDP`（fork 已用 `desktop-browser-cdp.ts` 的 `isDesktopCdpBrowserLive()`、URL 归一化等补丁吸收过一轮不稳定）。

---

## 附：关键文件索引

**自制**
`apps/desktop/src/{main,host-boot,host-boot-wire,boot-graph,spa-index,protocol-dsh,preload,ipc-contract,ipc-api-bridge,ipc-downlink,browser-view-manager,browser-view-bounds,browser-bounds-handler,browser-bounds-ipc,desktop-browser-cdp,exit-guard,single-instance,window-bounds,app-menu,app-branding,app-icon,attach,load-url,window-open-policy,packaging-paths,packaging-env,artifact-smoke}.ts`
`packages/bundle/desktop-app/cordis.patch.yml`、`packages/host/apiproxy/src/{browser-delivery,browser-registry,browser-page-scripts,browser-navigation-url}.ts`、`packages/client/connection/src/client/{ipc-api-client,ipc-bridge}.ts`、`packages/client/connection/src/ipc-channels.ts`、`packages/client/ui-browser/src/client/browser-desktop-occupant.ts`、`packages/browser/tool-browser/src/index.ts`、`apps/cli/src/{args,desktop-launcher,profile-boot}.ts`、`packages/boot/app-boot/src/profile.ts`、`apps/web/vite.desktop-dev.config.ts`、`scripts/{dev-desktop,prepare-desktop-packaging}.ts`、`.github/workflows/desktop-packaging.yml`
ADR/PRD：`docs/adr/0009-desktop-shell-electron-delivery.md`、`docs/adr/0010-desktop-browser-electron-cdp.md`、`docs/prd/desktop-v5.md`、`CONTEXT.md`

**上游 0.2.2**
`apps/desktop/src/{main,host-process,ipc,browser-guests,platform-view,project-manager,paths,core-package-set,runtime-tree,single-instance,quit-confirmation,tray,keyboard,locale,settings-file,command-installation,command-management}.ts`、`apps/desktop/electron-builder.config.mjs`、`apps/desktop/scripts/electron-builder-config.mjs`、`apps/desktop/{installer,renderer,resources,publishing,packaging}/*`、`apps/desktop-host/src/{index,cli,authentication}.ts`、`apps/desktop-publisher/src/*`、`packages/host/authentication-desktop/src/index.ts`、`packages/client/connection/src/{rpc,http-bridge,websocket-downlink}.ts`、`packages/client/web/src/apply-injections.ts`、`packages/client/ui-sidebar-browser/src/{types.ts,client/electron/*}`、`packages/bundle/web-app/cordis.patch.yml`、`packages/util/official-bundles/src/bundles.ts`、`docs/user/guide/desktop-uninstall-cleanup.md`、`docs/architecture.md#desktop-application`
