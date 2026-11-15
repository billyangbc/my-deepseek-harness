# 决策：本轮保留 CodeMirror，Monaco 延后（2026-11-14）

## 结论

本轮交付保留 CodeMirror 6，不将 Monaco 作为验收条件。Monaco 迁移需要先补客户端 CSS/worker/资产管线，不是编辑器组件替换。

## 依据

1. **构建形态**：树外 client 插件由 esbuild 打成单个 `client.js`，经 loader 的 `factory(require)` 注入；产物中没有独立 CSS 文件、worker 文件或字体资产。
2. **服务能力**：`/plugins` 路由只服务 `.svg/.png/.webp/.jpg` 资产（带内容修订），不能承载 JS worker 或 CSS。
3. **Monaco 的依赖**：Monaco ESM 入口携带主题 CSS、4 类 language worker 与字体；直接打包会得到无样式、无 worker 的编辑器，语言服务（TS/JSON 诊断）不可用。
4. **现有替代完整度**：CodeMirror 已覆盖编辑、保存、冲突检测、LSP 诊断/hover/定义跳转/补全、Markdown 预览、图片预览、文件操作；打包体积约 1.2 MB。
5. **成本与收益**：迁移需要改造客户端构建协议（CSS 抽取、worker 产物、loader 资产路由）或复用官方 client preset；收益主要是品牌一致性与更完整的 LSP 编辑体验，不影响本轮功能验收。

## 若后续必须迁移，PoC 范围

1. 在 `tooling/build-client.mjs` 中增加 CSS 抽取：esbuild `write:false` 收集 `.css` 输出，运行时注入 `<style>`。
2. worker 设计：将 worker 作为 **第二个 bundle** 输出，使用 `data:` URL 或新增资产路由类型；禁止依赖 `/plugins` 现有图片路由。
3. 语言服务：为 TS/JSON 注入 `MonacoEnvironment.getWorker`；LSP 面板仍走自有 `api-lsp`，与 Monaco 的语言 worker 不混用。
4. 验收：包体上限、首屏时间、诊断/补全/定义跳转与现有一致，卸载后无残留样式与 worker。

## 触发条件

- 产品要求像素级对齐上游 Monaco 编辑器；或
- 需要 Monaco 独有能力（多光标大文件、diff editor、内置 TS 语言服务）；或
- 官方 client preset 已可在树外复用资产/worker 管线。

在此之前，CodeMirror 是明确的产品选择，不是临时占位。
