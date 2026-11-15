# 未完成项归档（2026-11-15）

状态：**冻结归档**。以下事项不在当前交付范围内继续排期；满足“解锁条件”时再从归档恢复，并按现有替代方案保持产品可用。

## A. 依赖上游扩展点（不改上游则无法完成）

| ID | 事项 | 现状与替代 | 解锁条件 |
|---|---|---|---|
| A1 | Add to Chat 的 **composer 预填** | 已实现「发送到对话」：走现有 `ctx.conversation.send` 立即发送；不能把选区写入草稿 | 上游提供 composer 草稿插入 API，或提出 Agent Note/PR |
| A2 | 上游文件树 Git 徽章 | 上游 `ui-sidebar-files` 没有行装饰 slot；已在**编辑器自有文件列表**显示 Git 状态码 | 上游提供文件树装饰扩展点 |
| A3 | Monaco 编辑器 | 保留 CodeMirror；决策见 [`p5-monaco-decision.md`](../baseline/p5-monaco-decision.md) | 客户端构建支持 CSS/worker/资产，或复用官方 preset |
| A4 | Mermaid 按需加载/包体优化 | 全量内联约 7.3 MB；聊天渲染与 lightbox 可用 | loader 分包或资产管线支持 |

## B. 依赖环境、网络或凭据

| ID | 事项 | 现状与替代 | 解锁条件 |
|---|---|---|---|
| B1 | 桌面签名/公证分发 | 已构建 unsigned DMG（265 MB）、启动并挂载验证；跨机器分发会被 Gatekeeper 拦截 | Apple 开发者证书 + notary 凭据 |
| B2 | 桌面持久安装（/Applications） | 仅验证 DMG 可挂载与 app 可启动 | 可接受写入 `/Applications` 的验收环境 |
| B3 | Agent 浏览器工具回归 | provider/preset 配置文档已给（Playwright MCP/Stagehand）；未跑真实网页工具调用 | Chromium、网络、可用模型凭据 |
| B4 | 推送到 `NanGePlus` 原仓库 | 无写权限（403）；已改用 `billyangbc` 作为正式远端 | 原仓库 collaborator 权限 |

## C. 测试与工程设施

| ID | 事项 | 现状与替代 | 解锁条件 |
|---|---|---|---|
| C1 | 组件测试（Git/编辑器面板） | 以 provider 测试 + 全链路 UI E2E 覆盖 | 引入 vitest+jsdom 或 Playwright 组件测试 harness |
| C2 | 补全 UI 自动化门禁 | provider completion 用假服务器测试；真实 clangd 下 Ctrl+Space 偶发为空 | 稳定的 LSP 光标上下文 fixture |
| C3 | 大规模仓库虚拟列表 | 5k 文件 status 基线 46–48 ms；未做虚拟滚动 | 明确超大仓库目标后设计 |
| C4 | 外部文件变更自动重载 | 未实现；重新打开标签可刷新 | CodeMirror 文件观察 + 冲突处理设计 |

## D. 产品/清理决策

| ID | 事项 | 现状与替代 | 解锁条件 |
|---|---|---|---|
| D1 | 多 Language Server 路由 | 单 provider 单命令 + `languageIds` 门控；C/C++ 走 clangd | 设计多实例配置或按语言拆 provider |
| D2 | 旧线删除/只读说明 | 旧线保留 `legacy/custom-main-2026-11-14` 并推送；未写正式清理记录 | 确认旧线保留策略后补一节文档 |
| D3 | Mermaid i18n | 错误文案与 aria-label 仍中文硬编码 | 复用 Git/编辑器翻译模式 |
| D4 | Git 图形交互增强 | 已交付 ASCII 拓扑文本；未做节点点击跳转/泳道着色 | 解析图结构或换结构化 log 输出 |

## E. 归档时已确认完成的能力

- 基座零差异、插件独立安装/卸载（三 bundle 矩阵）
- Git：状态/diff/按块暂存/提交/push/ahead/历史详情/多远程/ASCII 图/zh-en
- 编辑器：文件操作、图片/Markdown、2 MB 守卫、LSP 诊断/hover/定义/补全、发送到对话、Git 状态标记、zh-en
- 终端：上游 Session 终端 E2E
- 浏览器：Web UI overlay 启用 + example.com iframe
- 桌面：unsigned 构建、启动、DMG 挂载
- 发布：基座 `custom-v0.2.0`，插件 `v0.3.0`；远端 `billyangbc`

## 恢复规则

1. 逐项检查“解锁条件”，而不是重新评估整个计划。
2. 优先恢复 A1/A2（上游扩展点）与 C1（测试设施）。
3. 恢复时更新本文件状态并从归档移出；相关实现文档同步更新。
