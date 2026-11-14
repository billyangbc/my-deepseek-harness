# S2b：装配层与工程配置定制明细（fork vs 上游 0.2.2）

- 本 fork：`<fork 工作区>`，`custom/main` HEAD `c5ac2488d9`（版本线 0.1.0-rc.5，分叉点 `47f943859b`）。
- 上游：`<upstream 工作区>` HEAD `39163dd30f`（`0.2.2-rc.3`）。
- 方法：只读代码与文档，未安装依赖/未构建。跨库 diff 需 `GIT_ALTERNATE_OBJECT_DIRECTORIES=<upstream>/.git/objects`；下文中 fork 路径相对 fork 根，上游路径相对上游根。
- 说明：分叉点在 0.1.0-rc.5，故大量「差异」是上游 0.2.2 的演进而非 fork 定制。判定「上游已具备」一律以上游 HEAD 为准；「改动性质」四类为：功能必需 / 只为让定制包能构建 / 品牌与流程偏好 / 上游已具备。
- 相关兄弟文档：`docs/custom/research/s3-desktop.md`（桌面壳取舍）、`s5a-upstream-ui.md`（UI 功能重叠）。

---

## 一、web-app bundle 定制明细与独立 bundle 替代方案

### 1.1 改动清单（`git diff 47f943859b HEAD -- packages/bundle/web-app`）

| 位置 | 改动 | 性质 |
|---|---|---|
| `packages/bundle/web-app/cordis.patch.yml:76-138` | 新增 `lsp`、`lsp-editor`、`lsp-stdio` 三行；`lsp-stdio` 配置 ts/json/html/css/yaml/python（`npx`）与可选 `gopls`/`jdtls` | 功能必需（编辑器诊断），包本体是 fork 新增 |
| `packages/bundle/web-app/cordis.patch.yml:220` | `connection` 行 `inject` 由 `[webRuntime]` 改为 `[webRuntime, webServer]` | 功能必需（等 webserver 就绪后再绑网关） |
| `packages/bundle/web-app/cordis.patch.yml:263-273` | 新增 `ui-file-editor`、`ui-git`、`ui-terminal`、`ui-browser` 四个 `dsh.client` 行 | 功能必需（DetailsPanel 工具箱） |
| `packages/bundle/web-app/package.json:58-61` | 四个 `dsh-client-ui-*` 依赖 | 功能必需；也可只留在安装侧 |
| `packages/bundle/web-app/package.json:95-97` | `dsh-lsp`、`dsh-lsp-editor`、`dsh-lsp-stdio` 依赖 | 同上 |
| `packages/bundle/web-app/package.json:111` | `@deepseek-ai/dsh-tool-browser` 依赖（本 bundle 无对应行，服务的是 preset 行） | 冗余/间接依赖 |
| `packages/bundle/web-app/README.zh.md:27` | 增加 LSP 默认服务器与 Go/Java 可选安装说明 | 文档 |
| `packages/bundle/desktop-app/cordis.patch.yml:77-83, 212-221, 447` | 同一批 `lsp`/`ui-*`/`agent-presets` 行在 desktop bundle 里再复制一份（两文件 450 vs 498 行） | 装配层最大冗余 |

### 1.2 层叠与 override 规则（决定「能否搬走」）

- 顺序：各 bundle patch 按 `dsh.profile.bundles` 顺序 → profile 自身 `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml` → 各 `--patch`（上游 `docs/user/develop/basic/publish.md:124-136`）。
- `insert` 无 `id` 时追加到列表尾部（`vendor/include/src/index.ts:93-95`），并立刻为后续层建索引（同文件 `:96-102`），所以后一层可以配置/禁用前一层插入的行。
- 后一层可按 `id` 覆盖已有行，但**整份替换 `config`**，不深合并（`publish.md:131-134`；`vendor/include/src/index.ts:105-130`）。
- 结论：`insert` 型新行天然可由独立层提供；改既有行（如 `connection.inject`）也能，但必须重述整行。

### 1.3 逐项替代判断

| 项 | 独立 bundle patch 可否替代 | 说明与代价 |
|---|---|---|
| `lsp`/`lsp-editor`/`lsp-stdio` 三行 | **可以** | 纯新 id 插入；配置自包含。前提：`dsh-lsp-editor`（fork 新包）作为 npm 包随 bundle 分发，或由安装侧携带；`lsp-stdio` 的 `inject: ['fs','lsp','lspEditor','subprocess']` 是包内事实，非装配。 |
| `ui-file-editor`/`ui-git`/`ui-terminal`/`ui-browser` 四行 | **可以** | 均为新 id 的 `dsh.client` 行；client 插件表按包名扫描，行插入位置不影响 tab 顺序。 |
| `connection.inject` 增加 `webServer` | **可以** | 后一层整行重述（id + name + inject + config）。注意上游 0.2.2 已把该行改为 `inject: [webStartup]` 且 `trustedHosts: !!js ctx.webStartup.trustedHosts`，迁移时应直接采用上游写法。 |
| `package.json` 新增依赖 | **可以** | 搬到新 bundle 的 `dependencies`；`tool-browser` 例外：它被 preset 行消费，应放在提供 preset 的层或安装侧。 |
| `README.zh.md` 段落 | **可以** | 随包文档。 |
| `ui-conversation` 的五段 tab 外壳 | **不可以** | tab 写死在客户端壳：`packages/client/ui-conversation/src/client/skeleton/DetailsPanel.tsx:17-24`（`SEGMENTS`）、`details-tab-labels.ts:9-15`、`contract/views.ts:10`（`DetailsTab`）。树外 bundle 只能填 `conversation.details.{editor,git,terminal,browser}` slot，不能增删段位；要么改壳（fork 现状），要么改用上游 `ctx.sidebarRightTabs` 注册制（见 S5a）。 |
| desktop/web 两份重复行 | **可以（推荐）** | 抽一个只含 `lsp*` + `ui-*` 的 toolbox bundle，profile 列 `[base, surface, toolbox]`；两 surface bundle 各删这批行。冲突面从两个 500 行文件收敛到一个。 |

### 1.4 建议形态与代价

- 新建 `@deepseek-ai/dsh-toolbox`（`dsh.bundle.patch: ./cordis.patch.yml`），只保留 1.1 中的 `lsp*`、`ui-*` 插入行与依赖；`web-app`/`desktop-app` 各自删除对应行。
- 该层必须排在 surface 层之后，因此 desktop 模板/初始化列表要显式加它（模板改动仍落在 profile 注册处，见第二节）；web 侧用户可用 `dsh plugin --profile web add` 追加，或由安装侧默认列表携带。
- 代价：toolbox 仍依赖 fork 包名与服务名；上游 0.2.2 的同类能力是 `client/ui-sidebar-files|terminal|browser`（web-app patch `:326-335`）与 `browser-use` 系列，rebase 时要么改名对齐上游、要么保留 fork 包并长期维护（S5a 已给出取舍）。
- 收益：`packages/bundle/web-app/cordis.patch.yml` 从 498 行降到约 380 行且不再触及 fork 专属包名，上游同文件冲突显著减少；`desktop-app` 同理。

---

## 二、profile 模板与 desktop profile

### 2.1 fork 现状

- `packages/boot/app-boot/src/profile.ts:113-117`：`PROFILE_TEMPLATES` 为 `Record<string, readonly string[]>`，含 `web`、`headless`、**新增** `desktop: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-desktop-app']`（`:116`）。
- 使用点：`loadProfile` 首次初始化（`:377-385`）、`normalizeShippedProfile`（`:300`）。desktop 启动路径：`apps/desktop/src/host-boot.ts:50` `runProfile({ profile: 'desktop', patchFiles: [], args: [] })`——依赖该模板 + `$DSH_HOME/profiles/desktop`。
- 附带改动：`apps/cli/README.md:39`、`apps/cli/README.zh.md:39` 把 `dsh-desktop-app` 写进「安装侧解析」的 bundle 列表。

### 2.2 上游 0.2.2 现状

- 模板已迁到 `packages/util/official-bundles/src/bundles.ts:9-27`，类型改为 `ProfileTemplate { bundles }`；只有 `acp`、`web`、`headless`、`sdk`、`sdk-minimal`，**没有 desktop**；另见同文件 `:29-35` 的 `INSTALLATION_OWNED_PROFILE_TUPLES`。
- 上游明确把 desktop 设计为「应用自有 profile」：`packages/boot/app-boot/README.md:52` 写明 Electron 的保留 Desktop profile 用 `loadProfileDirectory` 加载已初始化目录，「不经 CLI profile 查找」。
- 实现：`apps/desktop/src/project-manager.ts:31` 取 `PROFILE_TEMPLATES.web`，`:141`/`:166` 写 `dsh.profile.bundles = [...base, ...web-app]`，`:174` `initProfile(projectDir, WEB_PROFILE.bundles)`；`apps/desktop-host/src/index.ts:46-49` `loadProfileDirectory(...)` 后以 `resolvedProfile` 传给 `runProfile`。`apps/cli/src/args.ts:83-87` `rejectElectronProfile()` 直接拒绝 `dsh --profile desktop`。
- 上游 `profile-boot.ts:232,331` 的 `RunProfileOptions.resolvedProfile` 正是为这种应用自有 profile 增加的口子。

### 2.3 能否树外新增 profile（不改该文件）

- 能建：`@deepseek-ai/dsh-app-boot` 的 `initProfile`（fork `packages/boot/app-boot/src/profile.ts:153`，上游同名函数在 `:198`）、`dsh plugin --profile <name> ...`（初始化列表为默认 `base`）或手写一个含 `dsh.profile.bundles` 的 `package.json`，都能创建任意名字的 profile；`dsh --profile <name> --from-default-profile web` 可从任一 shipped 模板复制到自定义名下。
- 不能注册「模板」：`PROFILE_TEMPLATES` 是编译期常量，被 `boot/plugin-manager/src/operations.ts:604`、`experimental/liveshare`、`experimental/webworker-packer` 消费；新增一个名字只能改上游该文件（fork 则是 `app-boot/profile.ts`）。
- 因此「desktop 模板」不是树外可加的扩展点，而是 fork 为自家启动器改的安装侧注册表。

### 2.4 结论

- desktop 模板属于 fork 自用 boot 路径的功能必需项，但**上游已用更完整的设计取代**（应用自有项目 + `resolvedProfile`）。迁移时应删模板、删 `desktop` 相关 README 描述与 `apps/cli/tests/built-bin.e2e.ts:724-738` 的 dump 断言，把 `apps/desktop/src/host-boot.ts` 改成「初始化/加载自有项目后传 `resolvedProfile`」。
- 若短期保留，模板只能写在 fork 的 `profile.ts`，无法用 bundle 消除；长期看它是与上游 `desktop` 保留名冲突的注册项（S3 §5.2-8）。

---

## 三、apps/cli 改动

### 3.1 改动明细与替代

| 位置 | 改动 | 性质 | 上游现状 / 树外替代与清理 |
|---|---|---|---|
| `apps/cli/src/args.ts:47-55` | 新增 `DesktopInvocation` 并入 `DshInvocation` 联合 | 功能必需（fork 启动器） | 上游无 `desktop` 子命令，反而保留并拒绝该 profile（`args.ts:83-87`）。无树外替代：子命令注册在 `parseDshArgs` 内，属启动器；迁移即删除。 |
| `apps/cli/src/args.ts:78,179-188` | help 增一行 + `program.command('desktop')`（`allowUnknownOption`/`passThroughOptions`，转发内层参数） | 同上 | 同上。上游 `dsh web`/`headless` 等是同一套 `resolveBoot`；桌面改由 `pnpm run dev:desktop` / `start:desktop`（根 `package.json` scripts，走 `@deepseek-ai/dsh-desktop`）。 |
| `apps/cli/src/bin.ts:50-54` | `case 'desktop'` 动态 import `spawnDesktop` 并 `process.exit` | 同上 | 迁移即删除。 |
| `apps/cli/src/desktop-launcher.ts`（新增 65 行） | 解析 electron 可执行文件、`apps/desktop/lib/main.js` 优先 / `src/main.ts` 回退、`spawn` | 功能必需（fork 壳） | 上游 `apps/desktop/scripts/dev.ts` 自己负责构建与启动；本文件无树外替代（安装侧启动器代码），可整体丢弃。 |
| `apps/cli/src/profile-boot.ts:34-39` | `SHIPPED_PRESET_ROOT` 增加「`../config` 不存在则 `../../config`」回退 | 只为让定制包能构建（`apps/desktop/src/host-boot.ts:8` 从 `../../cli/lib/types/profile-boot.js` 导入，深度不同） | 上游 `profile-boot.ts:78` 仍是单锚点；上游已无 preset 根注入。清理：改用 `resolvedProfile` 入口后回退与 preset 注入一并删除。 |
| `apps/cli/src/profile-boot.ts:57-64` | `INSTALL_ANCHOR` 增加两级回退，找不到则抛错 | 同上 | 同上游单锚点；上游 app-boot 负责 `installAnchor` 解析。删除。 |
| `apps/cli/src/profile-boot.ts:167-176` | 组合后把 `agent-presets` 行 `config.roots` 覆写为 `[{ path: SHIPPED_PRESET_ROOT, trust: 'system' }]` | 功能必需（fork 目录式 preset） | 上游 preset 是 bundle patch 内的声明式 row（见 3.2），启动器不再注入根。删除后由 bundle 承载。 |
| `apps/cli/package.json:76,79,101` | 增 `tool-browser`、`desktop-app` 依赖与 `electron` devDependency | 功能必需 | 上游 `apps/cli` 依赖列表不含 desktop/browser/electron（清单里相关项为 0）；桌面依赖在上游 `apps/desktop`。迁移即删除 `desktop-app`/`electron`，`tool-browser` 视 preset 方案。 |
| `apps/cli/tsconfig.json:35-37` | project reference 指向 `packages/bundle/desktop-app` | 只为让定制包能构建 | 上游 face 配置手工维护，无自动生成；包移出后删除该 ref。 |
| `apps/cli/config/agent-presets/*/agent.cordis.yml` 末尾（standard:253、code:254、cordis:241） | 三个 preset 各加 `- id: tool-browser / name: '@deepseek-ai/dsh-tool-browser'` | 功能必需 | 见 3.2：上游无此目录；替代要么整份重述 preset row，要么按上游可选 bundle 模式（`disabled: !!js "!ctx.get('…')"`）。 |
| `apps/cli/tests/built-bin.e2e.ts:724-738`、`tests/desktop-launcher.spec.ts` | desktop dump 断言、启动器解析断言 | 只为定制测试 | 随命令删除。 |
| `apps/cli/README.md:39`、`apps/cli/README.zh.md:39` | bundle 解析列表加 `dsh-desktop-app` | 文档 | 随模板删除。 |

### 3.2 preset 结构差异（决定工具行能否树外添加）

- fork：目录式。`apps/cli/config/agent-presets/<id>/agent.cordis.yml` + `preset.yml`，由启动器把根注入 `agent-presets` 行（3.1）。多根按顺序发现、同 id 先到者胜（`packages/preset/agent-presets/src/discovery.ts:174`、`src/index.ts:96-104`）。因此加一行工具只需在对应 YAML 末尾追加两行。
- 上游 0.2.2：声明式 row。四个 preset 是 `packages/bundle/web-app/presets/{standard,ptc,minimal,cordis}.patch.yml`，由该 bundle 的 `dsh.bundle.patch` 数组按序应用（`packages/bundle/web-app/package.json:49-59`）；`apps/cli/config/agent-presets/**` 已删除。
- 关键约束：`applyEntryPatches` 只索引顶层与 `group: true` 的 `config` 数组（`vendor/include/src/index.ts:66-75`），**不索引 preset row 的 `config.plugins`**。所以树外要给 shipped preset 加工具行，只能（a）整份重述该 row 的 `config.plugins`（web-app 自己的注释也说明编辑结果按此覆盖，`presets/standard.patch.yml:1-3`），或（b）让工具行常驻 shipped preset、用 `disabled: !!js "!ctx.get('…')"` 在服务缺席时关掉（上游 `experimental/plugin-market-bundle/cordis.patch.yml` 注释即此模式），bundle 只提供服务。
- 结论：fork 现在这种「目录 + 两行 YAML」在上游结构里不存在；迁移时 `tool-browser` 的装配成本从 3 行升为「重述 3 份 preset 的 plugins 列表」或改造成可选 bundle 模式。这是本次盘点中最不划算的树外化项，建议等上游接受「preset 级插件注入」扩展点再动。

---

## 四、工程配置改动表

| 文件（行） | 改了什么 | 性质 | 树外替代 | 清理方式 |
|---|---|---|---|---|
| `package.json:137-139` | `web`/`start`/`pack:dsh` 脚本 | 品牌与流程偏好（入口别名；`pack:dsh` 近似上游 `release:pack`） | 用户可自行 `npm run dsh -- web` | 直接删；保留 `dsh`，或对齐上游 `start:web` |
| `package.json:145-149` | `dev:desktop`、`prepare:desktop-packaging`、`pack:desktop[:dir]`、`smoke:desktop-packaging` | 功能必需（fork 壳） | 无（启动器/打包入口）；上游对应 `dev:desktop`/`package:desktop:*` 走 `@deepseek-ai/dsh-desktop` | 随 `apps/desktop` 迁移删除 |
| `pnpm-workspace.yaml:42` | `allowBuilds.electron: true` | 只为让定制包能构建（electron@35 有 postinstall） | 无（安装期安全策略） | 迁到上游 electron@44 后删除：上游 lock 中 `electron@44.7.0` 无 `requiresBuild`，故无需该条 |
| `pnpm-workspace.yaml:57` | `electron-winstaller: true` | 品牌与流程偏好（允许 Squirrel 辅助；Windows 走 NSIS 时不需要） | 无 | 恢复上游 `false` 并删除该行 |
| `tsconfig.base.json:198-200` | 3 个显式 `paths` 别名（ui-file-editor/git/terminal） | 只为让定制包能构建（wildcard `@deepseek-ai/dsh-*` 已覆盖大部分；client 组需显式） | 包移出后不需要 | 删除；上游 `scripts/gen-tsconfig-paths.ts` 会自动生成映射区（`tsconfig.base.json:439-646`） |
| `tsconfig.host.json:234,246,296` | `tool-browser`、`desktop-app`、`lsp-editor` project reference | 只为让定制包能构建 | 包移出后不需要 | 删除；face 聚合引用上游仍需手工维护，但只针对保留下来的包 |
| `tsconfig.client.json:68-71` | 4 个 client UI project reference | 同上 | 同上 | 同上 |
| `vitest.config.ts:16-31,137,148` | monaco/xterm CSS/AMD 的全局 alias stub（`monaco-editor.stub.ts`、`xterm.stub.ts`、`xterm-css.stub.ts`） | 只为让定制包能测试 | 定制 UI 包移出后自带测试配置 | 删除 alias 与 3 个 stub 文件；上游无同类 alias（`testingAliases` 只做 claude-code/generated-remote） |
| `.gitignore:24,35` | 忽略 `.cursor/`、`apps/web/public/material-icons/` | 环境偏好 / 生成物 | 生成物可选 | 第一条可删；第二条随 `ui-file-editor` 的图标同步脚本删除 |
| `apps/web/package.json:23` | `prebuild` 调 `packages/client/ui-file-editor/scripts/sync-material-icons.mjs` | 只为让定制包能构建（跨包预构建耦合） | 定制 UI 包自己产资产，app/web 不感知 | 删除 `prebuild` 与生成的 `public/material-icons/` |
| `apps/web/vite.config.ts:16` | `DSH_DESKTOP_DEV=1` 时放行 `serve`（旁路 standalone 拒绝） | 功能必需（fork 桌面 dev） | 上游 `apps/desktop/scripts/dev.ts` 直接启动真实 host，不跑 Vite dev | 删除旁路，恢复无条件拒绝 |
| `apps/web/vite.config.ts:52` | `mermaid` 进 vendor chunk | 功能必需（fork 的 Mermaid 渲染） | 若 Mermaid 能力上推上游则成为上游配置 | 能力删除后删该行；上游无 mermaid 依赖 |
| `apps/web/vite.desktop-dev.config.ts`（新增 122 行） | 读 `.sessions/desktop-boot-graph.json` 注入 `window.__DSH_BOOT__`，并从磁盘直供 `/plugins/*` | 功能必需（fork 桌面 dev） | 无（上游 dev 流程不同） | 与 `scripts/dev-desktop.ts` 一起删除 |
| `apps/web/tsconfig.json:39` | include 新增 `tests/details-segmented-tab.e2e.ts` | 只为定制测试 | 测试随 UI 包移出 | 删除测试与 include |
| `apps/web/tests/scaffold.ts:817` | 增加 `{{shell-profile}}` 归一化 | 品牌与流程偏好（掩盖终端 tab 标题的平台差异） | 无 | 上游要求修 fixture 而非 normalizer（AGENTS.md 测试策略）；随终端 tab 移除 |
| `apps/web/tests/snapshots/lifecycle-chrome/*.expected.md` | 各删 1 行（五段 tab 引入后的 chrome 变化） | 功能必需（快照跟随壳改动） | 无 | 恢复上游快照 |
| `apps/web/tests/snapshots/details-segmented-tab/*`（5 个新快照） | 工具箱五段 tab 的金样本 | 同上 | 随测试删除 | 删除目录 |
| `scripts/client-bundle-css.spec.ts:53-73` | 新增「plain `.css` 不走 CSS Modules」测试 | 只为让定制包能构建 | 包移出后不需要 | 可保留（纯增益）或随 `tsdown.client.ts` 改动一起删；上游 `tsdown.client.ts:355,444` 已原生支持 plain CSS |
| `packages/client/tsdown.client.ts:229-244,269` | `.module.css` 判定放宽为 `.css`、plain CSS 不 hash、`codeSplitting: false` | 功能必需（monaco 的 `.css` + 动态 import） | 无 | 若保留 Monaco，需在上游 tsdown 管线重新落地；上游已支持 plain CSS，只差 `codeSplitting` 语义 |
| `scripts/agent-note-tree.ts:25` | `ROOT_ALLOWLIST` 去掉 `CLAUDE.md`（fork 删了该符号链接） | 品牌与流程偏好 | 无 | 恢复 `CLAUDE.md` 或保留 fork 的删除策略；与上游强制冲突 |
| `scripts/build-exe-for-python-sdk.ts:34` | `DEPLOY_ONLY_DOCS` 改为 `['README.md','README_EN.md']` | 品牌偏好（README 命名颠倒：中文为 `README.md`、英文 `README_EN.md`） | 无 | 与上游 `README.md/README.zh.md` 二选一；改名即回退 |
| `scripts/translation-pairing.manifest.json:10-11` | 排除 `README.md`/`README_EN.md` | 品牌偏好 | 无 | 随 README 命名方案回退 |
| `scripts/verify-translation-prompt.ts:28-29` | 示例对里删掉 README 对 | 品牌偏好 | 无 | 回退 |
| `scripts/verify-package-readme-model-experience.ts:74-76` | 3 个定制 UI 包的 `none` 说明 | 只为让定制包能构建（readme 门禁） | 包移出后不需要 | 随包删除 |
| `scripts/check-workspace-constraints.ts:137` | `dsh-desktop-app` 允许发布 `cordis.patch.yml` | 只为让定制包能构建 | 上游同表已有 base/web-app/headless；新 bundle 加入口即同类一行 | 保留（若保留 bundle）或删除 |
| `scripts/{dev-desktop,prepare-desktop-packaging,smoke-desktop-packaging}.ts`、`run-electron-builder.mjs` | 开发/打包/冒烟入口 | 功能必需（fork 壳） | 上游 `apps/desktop/scripts/*` 更完整（签名、更新通道、runtime 预置） | 随壳迁移删除 |
| `scripts/repair-session-log.ts` | 一次性修复重复 seq 的运维脚本 | 流程偏好 | 无 | 可保留（与装配无关） |
| `tsconfig.base.json:231-284`（wildcard 组列表） | **缺** `./packages/browser/*/src`；`dsh-client-ui-browser` 也无显式别名（当前仅自测相对导入，未暴露） | 缺陷（clean-tree 源解析缺口） | 上游生成器按 `packages/*/*` 探测，不会有此缺口 | rebase 后由生成器覆盖；短期补 wildcard 组 |
| `.github/workflows/` 12 个既有文件 | 删除 `push`/`pull_request` 触发，改 `workflow_dispatch`（`ci.yml:3-4`、`e2e.yml:17-20`、`docs-pages.yml:3-4`、`release.yml:9-10`、`sandbox.yml:10-11`、`expected-filenames.yml:3-4`、`issue-policy.yml:3-4`，以及 build-exe/release-vendor/landlock-run/issue-lifecycle/python-release 的 PR 半区） | 流程偏好（有决策记录：`.agents/notes/implemented/process/2026-08-20-disable-push-pr-ci.md`） | 无（这是仓库设置偏好，非代码扩展点） | rebase 后必须重新施加；否则上游 PR 触发会把整条矩阵跑起来 |
| `.github/workflows/desktop-packaging.yml`（新增 74 行） | PR/手动触发、mac+win 未签名安装包 + 冒烟 | 功能必需（fork 打包验证） | 上游 `desktop-publish.yml`/`desktop-rollout.yml` 走发布/签名流程，不是 PR 打包 | 随打包脚本删除 |

---

## 五、结论

1. **能用独立 bundle 层消除的**：web-app 里全部 `lsp*` 与四个 `ui-*` 插入行、`connection` 行重述、对应依赖与 README 说明。层叠规则（`publish.md:124-136`、`vendor/include/src/index.ts:80-102`）明确支持后层追加与按 id 覆盖。
2. **必须改上游文件、bundle 消除不了的**：`ui-conversation` 的五段 tab 壳（`DetailsPanel.tsx:17-24`）、`profile.ts` 的 `desktop` 模板（`profile.ts:116`）、`apps/cli` 的 `desktop` 子命令/启动器、目录式 preset 的根注入（`profile-boot.ts:167-176`）。这些属于「壳与启动器」，不是装配。
3. **desktop 模板应删除而非移植**：上游 0.2.2 已用「应用自有项目 + `loadProfileDirectory` + `runProfile.resolvedProfile`」实现同一目标（`app-boot/README.md:52`、`apps/desktop/src/project-manager.ts:141,174`、`apps/desktop-host/src/index.ts:46`），并显式拒绝 `dsh --profile desktop`（`apps/cli/src/args.ts:83-87`）。保留它等于与上游争同一 profile 目录。
4. **装配层最大冗余是 web/desktop 两份 patch 重复**（`lsp`/`ui-*`/`agent-presets` 行各一份，450 vs 498 行）。抽 `toolbox` bundle 后，两者都缩到「surface 差异 + 引用」，rebase 冲突面从两份整文件收敛到一份小文件。
5. **preset 工具行是最不划算的树外化项**：上游 preset 是 web-app bundle 内的声明式 row，`config.plugins` 不能被 patch 局部追加（`applyEntryPatches` 只索引 group 数组），树外只能整份重述或用「常驻行 + `disabled: !!js !ctx.get(...)`」模式。建议先推动上游提供 preset 级插件注入扩展点，再迁移 `tool-browser` 行。
6. **工程配置里大半是「镜像改动」**：`tsconfig.*` 的 3 处 paths + 8 处 project reference、`vitest.config.ts` 的 3 个 stub alias、3 个 readme 门禁条目，都是为让 fork 新包通过构建/测试。上游 0.2.2 用 `gen-tsconfig-paths` 生成映射区，rebase 后手写别名应删除、交给生成器。
7. **两处配置应回退**：`electron-winstaller: true`（上游 `false`，Windows 走 NSIS）与 `DSH_DESKTOP_DEV` 旁路 standalone 拒绝（上游桌面 dev 不跑 Vite dev）。二者都不影响 fork 功能，但会扩大安装脚本面与 dev/prod 差异。
8. **品牌/流程偏好可整体隔离**：README 命名（中文 `README.md` + `README_EN.md`）连带 3 个校验脚本与 1 个构建脚本；`CLAUDE.md` 删除连带 `agent-note-tree.ts:25`。这些与产品能力无关，若目标是「薄 fork」，应统一改回上游命名，只保留 `CUSTOM.md`/`CONTEXT.md` 一类新增文件。
9. **CI 触发关闭是显式决策，不是可删噪音**（`2026-08-20-disable-push-pr-ci.md`）。rebase 后必须逐文件重放，否则上游 `pull_request` 触发会重新激活整条矩阵；反之若改用上游 CI，需同时恢复这 12 个既有工作流（新增的 `desktop-packaging.yml` 另行处置）。
10. **发现一个 clean-tree 缺陷**：`tsconfig.base.json` 的 wildcard 组列表缺 `./packages/browser/*/src`，`@deepseek-ai/dsh-tool-browser` 的自测按包名自引用、只能经 `node_modules`/`lib` 解析，违反「静态门禁走 paths 到 src」的规则（`ui-browser` 当前自测走相对导入，暂未暴露）；补组或等上游生成器覆盖。
