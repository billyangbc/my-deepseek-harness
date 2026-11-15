# 二次开发分析与改造文档索引

本目录是「升级上游基线 + 二次开发解耦 + 功能插件化」的决策与计划文档，面向维护者。产品行为说明仍在根目录 [CUSTOM.md](../../CUSTOM.md)。

| 文档 | 内容 | 适用场景 |
|---|---|---|
| [01-现状分析.md](./01-现状分析.md) | 基线规模、二次开发清单、耦合点、上游 0.2.2 结构变化、能力对照、卸载残留、风险 | 了解「为什么必须重基线、为什么必须插件化」 |
| [02-解耦改造方案.md](./02-解耦改造方案.md) | 目标架构、仓库/分支模型、功能处置决策、插件包设计、seam 缺口、安装与卸载契约、版本策略、验收门禁 | 评审方案、确认产品决策 |
| [03-实施计划.md](./03-实施计划.md) | 路线选择、阶段与任务、时间线、门禁、降级路径、里程碑 DoD、立即执行清单 | 排期与执行 |
| [research/s2b-host-integration.md](./research/s2b-host-integration.md) | 装配层与工程配置逐项清单：bundle patch、profile 模板、CLI、preset、tsconfig、CI、品牌脚本 | 装配与工程配置清理 |
| [research/s3-desktop.md](./research/s3-desktop.md) | 桌面壳 V5 与上游桌面应用的逐项对比、独有能力、安全与生命周期缺陷、迁移建议 | 桌面决策 |
| [research/s4-tools-lsp.md](./research/s4-tools-lsp.md) | Agent 工具、LSP、Markdown/预览、20 项官方包零散补丁的逐项结论 | 工具/LSP/UI 迁移与上游 PR 选题 |
| [research/s5a-upstream-ui.md](./research/s5a-upstream-ui.md) | 上游 UI 能力与 fork 定制的映射（文件/终端/浏览器/Markdown/LSP/侧栏扩展点） | UI 迁移与扩展点选择 |
| [baseline/2026-11-14.md](./baseline/2026-11-14.md) | P0/P1 基线：归档引用、安装/构建/类型/测试结果与本机已知失败 | 基线与环境对照 |
| [baseline/p2-2026-11-14.md](./baseline/p2-2026-11-14.md) | P2 插件 SDK PoC：G1/G2 结论、E2E 证据、安装/卸载验收 | 插件工程开工 |
| [baseline/p3-2026-11-14.md](./baseline/p3-2026-11-14.md) | P3 Mermaid 插件：实现、真实浏览器渲染证据、包体与测试缺口 | 客户端插件复用 |
| [baseline/p4-2026-11-14.md](./baseline/p4-2026-11-14.md) | P4 Git 插件：核心操作、按块暂存、push、历史详情、硬约束、待办 | Git 插件续作 |
| [baseline/p5-2026-11-14.md](./baseline/p5-2026-11-14.md) | P5 编辑器：写能力、CodeMirror 面板、文件操作/图片/守卫、stdio LSP（诊断/hover）、新约束 | 编辑器续作 |
| [baseline/p6-2026-11-14.md](./baseline/p6-2026-11-14.md) | P6 切换：终端 E2E、官方浏览器/桌面行、cutover 清单、剩余验证 | 切换执行 |

## 一句话结论

当前 fork 依赖的三个官方结构（`host/apiproxy`、`ui-conversation` 详情栏、自制 Electron 壳）已被上游 0.2.2 整体替换；继续合并会留下深度耦合。决策已确认：冻结旧线、重基线到 `0.2.2-rc.3`，只保留 Git 面板、Monaco 编辑器、Mermaid 三项独有能力并以独立组合包交付；终端、浏览器、桌面改用上游官方能力；插件暂不发布 npm。

## 调研状态

| 主题 | 状态 | 产出 |
|---|---|---|
| 客户端 UI 定制耦合 | 已覆盖（01 §3.3、research/s5a） | 子代理未产出，关键结论已由主分析补齐 |
| Host RPC 与装配层耦合 | 已覆盖（01 §3.1/§3.2、research/s2b） | research/s2b-host-integration.md |
| 桌面壳 | 已完成 | research/s3-desktop.md |
| 工具/LSP/零散补丁 | 已完成 | research/s4-tools-lsp.md |
| 上游 UI 能力映射 | 已完成 | research/s5a-upstream-ui.md |
| 插件装载/卸载契约 | 已覆盖（02 §6、03 §5） | 以主分析为准，未单独成文 |

## 维护约定

- 本目录记录**当前决策与计划**；实施完成后，把「现状」类内容归档，把最终形态写回 `CUSTOM.md` 与插件仓文档。
- 上游升级时先更新 [01](./01-现状分析.md) 的基线与差异数字，再评估 [03](./03-实施计划.md) 的降级路径是否仍成立。
- 相对路径链接到上游仓库的引用一律写「上游 + 相对路径」，避免本地绝对路径入库。
