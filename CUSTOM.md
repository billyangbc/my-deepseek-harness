# 相对官方的定制说明

本 fork 已完成基线与解耦改造：`custom/main` 的产品代码与上游 `dsh-v0.2.2-rc.3` 基线**零差异**，全部定制以独立 DSH 插件的形式放在外部插件仓 [`NanGePlus/dsh-plugins`](https://github.com/NanGePlus/dsh-plugins)，按 profile 安装/卸载。

## 基线

- 上游仓库：<https://github.com/deepseek-ai/deepseek-harness>
- 基线提交：`39163dd30f703a79079e2d2446545cff3c453738`（`0.2.2-rc.3`，`upstream/develop`）
- 本 fork 分支：`custom/main`（= 基线 + 本文件与 `docs/custom/`）
- 本 fork 远端：<https://github.com/billyangbc/my-deepseek-harness>（`refactor/main` 同步；旧线见 `legacy/custom-main-2026-11-14`）
- 插件仓远端：<https://github.com/billyangbc/dsh-plugins>（tag `v0.2.2`）
- 旧定制线：标签 `legacy/custom-main-2026-11-14`（只读留档，可回滚）
- 零差异范围：`packages/`、`apps/`、`vendor/`、`package.json`、`pnpm-lock.yaml`、`pnpm-workspace.yaml`、`tsconfig*.json`

## 定制能力（插件）

| 能力 | Bundle | 说明 |
|---|---|---|
| Mermaid 聊天图表 | `@nange/dsh-plugin-mermaid` | 围栏渲染、缩放进/平移 lightbox、主题跟随 |
| Git 面板 | `@nange/dsh-plugin-git` | 状态、diff、按块 stage/unstage/discard、提交、push/ahead、历史与提交详情 |
| 编辑器 | `@nange/dsh-plugin-editor` | 文件树（含 Git 状态标记）与新建/重命名/删除、CodeMirror 编辑、图片与 Markdown 预览、2 MB 守卫、发送到对话 |
| LSP | 同上 | clangd 诊断、hover、F12 定义跳转、补全 |

能力到插件的分层：`dsh-<cap>` 定义 → `dsh-<cap>-local` provider → `dsh-api-<cap>` Remote → `dsh-client-ui-*` 面板 → `dsh-plugin-*` bundle。

## 安装

```sh
# 在插件仓构建（需要匹配版本的 DSH checkout）
DSH_CHECKOUT=/path/to/deepseek-harness pnpm run build

# 安装到某个 profile
dsh plugin --profile <profile> add /absolute/path/to/packages/plugin-git
dsh plugin --profile <profile> add /absolute/path/to/packages/plugin-editor
dsh plugin --profile <profile> add /absolute/path/to/packages/plugin-mermaid
```

编辑器 bundle 默认把 C/C++ 交给 `clangd`；其他语言在 profile 的 `nange-lsp` 行改 `command`/`languageIds`。

## 升级

1. 在基座仓切换新的上游提交，确认 `tooling/verify-zero-diff.mjs` 通过。
2. 更新插件仓的 DSH 依赖版本与 `tooling/generate-via-checkout.mjs` 的基线，重新生成 Typert artifacts。
3. 重新构建三个 bundle，按「卸载旧 bundle → 安装新 bundle → 启动」顺序升级。

## 卸载

```sh
dsh plugin --profile <profile> remove @nange/dsh-plugin-git
dsh plugin --profile <profile> remove @nange/dsh-plugin-editor
dsh plugin --profile <profile> remove @nange/dsh-plugin-mermaid
```

`--dump-config` 不再出现 `nange-*` 行；`link:` 开发安装可能在 `node_modules/@nange/` 留下符号链接，移除即可。

## 验证与文档

- 计划与实施记录：[`docs/custom/`](docs/custom/README.md)
- 逐项审计：[`docs/custom/baseline/plan-audit-2026-11-14.md`](docs/custom/baseline/plan-audit-2026-11-14.md)
- 未完成项归档：[`docs/custom/archive/unfinished-2026-11-15.md`](docs/custom/archive/unfinished-2026-11-15.md)
- 零差异脚本：插件仓 `tooling/verify-zero-diff.mjs`；CI 为本地校验（远端 Actions 已关闭，workflow 仅手动触发）
- UI E2E：Mermaid/Git、编辑器保存、文件操作、LSP、终端、Markdown 预览

## 已知限制

- 构建期需要匹配的 DSH checkout（Typert 生成 G1 降级方案）；运行时安装是树外。
- 客户端构建仍是单 JS bundle，暂不支持 CSS Modules/worker/资产；因此编辑器用 CodeMirror 而非 Monaco。
- Mermaid 全量内联（约 7.3 MB），Git 历史为线性列表，编辑器上限 2 MB。
- 「发送到对话」使用现有 `ctx.conversation.send`，是立即发送；把选区预填进 composer 仍需上游插入 API。
