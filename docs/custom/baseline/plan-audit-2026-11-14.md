# 计划实施审计（2026-11-14）

审计对象：[03-实施计划.md](../03-实施计划.md) 的 P0–P7 任务表。依据：`docs/custom/baseline/` 记录、插件仓提交与 fixtures、当前分支与远端引用。

## 1. 总体结论

按计划人日加权，完成度约 **62%**（27/44 人日）。P0–P3 基本完成，P4/P5 主体可用但细节缺口明显，P6 只完成终端切换，P7 尚未开始。工程风险已从「能否解耦」转为「交付收尾与验收」。

| 阶段 | 估算 | 完成度 | 状态 |
|---|---:|---:|---|
| P0 冻结与准备 | 1 | ~90% | 标签/分支/基线完成，远端未更新、CI 策略未记 |
| P1 上游基线验证 | 2 | ~85% | install/build/typecheck/快照/基线完成；桌面冒烟未做 |
| P2 插件工程 PoC | 4 | ~85% | 插件仓、生成降级、client bundle、安装脚本完成；CI 缺失 |
| P3 Mermaid | 3 | ~80% | 渲染/主题/lightbox/安装完成；组件测试、快照、tag、i18n 缺失 |
| P4 Git | 10 | ~70% | 核心操作/按块/push/历史详情完成；Graph、徽章、多远程、i18n、性能、组件测试缺失 |
| P5 编辑器 | 14 | ~65% | 写能力/文件操作/图片/Markdown/LSP 诊断/hover/定义/补全完成；Monaco、Add to Chat、TipTap、外部变更重载、5 MB、测试矩阵缺失 |
| P6 切换 | 6 | ~45% | 终端采用上游并验收；浏览器/Agent 浏览器/桌面未验收；远端未推送 |
| P7 清理交付 | 4 | 0% | 全部未开始 |

## 2. 逐项核对

### P0（部分完成）

- P0-1：本地 `legacy/custom-main-2026-11-14`、`archive/custom-main-v1`、`archive/custom-2026-09-05` 均存在；远端只有 `archive/custom-2026-09-05`，旧 `custom/main` 仍指向 `c5ac2488d9`，新线未推送。
- P0-2：`upstream` remote 与 `base/upstream` 存在。
- P0-3： `docs/custom/baseline/2026-11-14.md` 存在。
- P0-4/5：已记录环境与回滚标签。
- P0-6：CI 触发策略与 README 回退决策没有单独的决策记录。

### P1（基本完成）

- 完成：install、build、typecheck、dump-config、snapshot、单元测试基线；5 个环境失败已记录。
- 缺失：P1-6 桌面冒烟未执行；P1-5 的 Web 冒烟由后续 E2E 间接覆盖，不是一次独立的 P1 验收。

### P2（基本完成）

- 完成：插件仓、Typert 降级生成、esbuild client bundle、`install-check.mjs`、`bundle-check.mjs`、G1/G2 结论记录。
- 缺失：**P2-5 CI 未创建**（插件仓无 `.github/`）；无 Node 24/26 矩阵。

### P3（大部分完成）

- 完成：围栏注册、流式回落、主题重绘、lightbox（缩放/平移/重置）、真实浏览器渲染、聊天内渲染（P4 补验）、安装/卸载。
- 缺失：P3-4 组件测试与页面快照；P3-5 插件仓 tag；i18n（中文硬编码）；7.3 MB 包体未优化。

### P4（主体完成）

- 完成：`status/diff/stage/unstage/commit/log/stageHunks/unstageHunks/discardHunks/push/commitDetail`（11 个操作）、Session 作用域、按块暂存、push/ahead、历史详情、差异着色、真实会话 E2E。
- 缺失：计划中的 13 操作尚有差距（文件级 discard、blame 等未定义）；提交**拓扑图**、提交详情外的 diff 语法高亮（现为行级着色）、Git 徽章、多远程管理、i18n、5k 文件性能记录；组件测试与 fast-check/快照；插件 README 未更新（仓库 README 仍只列 demo/mermaid）。

### P5（主体完成）

- 完成：写能力定义/provider/Remote（根内约束、哈希冲突）、CodeMirror 编辑器、文件列表与新建/重命名/删除、图片预览、2 MB 上限、Markdown 预览（marked+DOMPurify）、LSP 诊断/hover/定义跳转/补全、脏标签关闭确认。
- 缺失：Monaco（当前 CodeMirror）、Add to Chat（无 composer 插入 API）、TipTap、外部变更自动重载、5 MB 大文件策略、`patterns` 匹配文本文件、组件/快照测试、卸载审计；补全 UI 在 clangd 下自动化不稳定。

### P6（部分完成）

- 完成：终端采用上游并通过 E2E；`cutover/p6` 快进 `custom/main`；干净 HOME 安装三 bundle、0 激活失败、typecheck/build/6 组 E2E 通过。
- 缺失：P6-2 Web 浏览器启用（官方 `ui-sidebar-browser` 目前桌面专属，未在 Web profile 打开）；P6-3 Agent 浏览器（browser-use/MCP）未验证；P6-4 桌面端构建/安装/启动未做；P6-8 桌面回归未做。
- 说明：P6-6/P6-7 的「补丁清理」在零差异基线上无对象，但旧线仍在 `legacy/custom-main-2026-11-14`，尚未删除或归档说明。

### P7（未开始）

- P7-1 旧定制包/构建配置/CI 残留未清理（旧线仅打标签）。
- P7-2 **零差异门禁未自动化**（当前靠人工 `git diff --stat base/upstream`）。
- P7-3 卸载矩阵已通过（web profile；桌面 surface 未测）。
- P7-4 `CUSTOM.md` 未重写为索引（仍是旧线说明）。
- P7-5 插件仓 `v0.1.0`/`v0.2.0`、基座 `custom-v0.1.0` 已打本地 tag（未推送）。

## 3. 主要风险

1. **远端分叉**：`origin/custom/main` 仍是旧线，本地 `custom/main` 基于上游历史；推送需要 force-with-lease，且 `legacy` 标签未推送。任何推送前必须确认备份与 tag。
2. **缺 CI**：插件仓与基座仓都没有针对「零差异 + 插件安装 + E2E」的自动门禁，当前证据全部来自本机脚本。
3. **文档与入口滞后**：插件仓 README 未列 git/editor；`CUSTOM.md` 未切换；用户无法从仓库首页获知新架构。
4. **测试结构**：fixtures 中混有 12 个一次性 `e2e-inspect*.mjs`；补全 UI 不稳定；没有快照/组件测试。
5. **能力缺口**：Monaco/Add to Chat 依赖上游扩展点或客户端资产管线；桌面/浏览器验收依赖环境。
6. **偶发加载顺序**：Mermaid 在干净 HOME 曾出现一次 slot 未声明告警，重试后渲染；未根因定位。

## 3.1 建议执行结果（2026-11-14 晚）

- 建议 1（P7 底座）：新增 `tooling/verify-zero-diff.mjs`、`.github/workflows/ci.yml`、插件 README 重写、`CUSTOM.md` 重写、插件仓 `v0.1.0` tag；零差异脚本已本地验证（对基线 pass、对旧线 fail）。
- 建议 2（推送）：**受阻**。`gh`/SSH 身份均为 `billyangbc`，对 `NanGePlus/my-deepseek-harness` 推送返回 403；`refactor/main`、`legacy/custom-main-2026-11-14`、`cutover/p6` 已在本地就绪，等待仓库写权限或切换到有权限的身份后再推。
- 建议 3（P4 收口）：多远程 push、ASCII 拓扑图、README、卸载矩阵完成；徽章/i18n/组件测试/性能仍在待办。
- 建议 4（P5 决策）：保留 CodeMirror，Monaco 延后，见 `p5-monaco-decision.md`。
- 建议 5（P6 验收）：Web 浏览器 UI 已实测（overlay 启用 + example.com iframe）；桌面 `build` 通过（Electron v44.7.0），启动/安装未测。
- P7 增量：`uninstall-audit.mjs` 全矩阵通过（3 bundle 单独 + 组合，行/manifest 干净，仅 `link:` 符号链接残留）；插件仓 tag `v0.1.0`、`v0.2.0`；基座 tag `custom-v0.1.0`。

## 4. 建议的下一步（按优先级）

1. **P7-2/P7-4/P7-5 先行**：把零差异检查写成脚本并接入 CI；重写 `CUSTOM.md` 与插件 README；给插件仓打首个 tag。这些是交付可信度的底座。
2. **推送决策**：确认是否将 `custom/main` 强推到 origin，并推送 `legacy/custom-main-2026-11-14`；推送后更新远端默认分支与 README。
3. **P4 收口**：插件 README、组件测试、Git 徽章/多远程/i18n 中至少补 README 与错误文案；性能基线可后置。
4. **P5 决策**：明确 Monaco 是否必须；若必须，先做客户端 CSS/worker 资产管线 PoC，再决定是否替换 CodeMirror；Add to Chat 需要上游扩展点提案。
5. **P6 验收**：在可用桌面环境跑一次官方桌面端与浏览器能力；Web 侧明确不启用浏览器 UI 的产品说明。
