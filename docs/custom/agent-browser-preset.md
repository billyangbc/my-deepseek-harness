# Agent 浏览器 preset 配置（2026-11-14）

本文说明在**不改上游代码**的前提下，通过 profile/agent preset 组合启用 Agent 浏览器能力。人类侧浏览器 UI 与 Agent 侧 browser-use 是两套能力，前者见 P6 记录。

## 1. 组成

| 行 | 作用 |
|---|---|
| `@deepseek-ai/dsh-browser-use` | 共享注册服务 `ctx.browserUse`，只允许一个 provider |
| `@deepseek-ai/dsh-experimental-browser-use-playwright-mcp` | Playwright MCP provider（推荐先试） |
| 或 `@deepseek-ai/dsh-experimental-browser-use-stagehand-native` | Stagehand 原生 provider，需要 OpenAI/Anthropic 等原生模型 key |

两个 provider 不能同时启用；MCP provider 会为每个新 Agent 初始化一个 MCP client，浏览器随 Session 存活，Session 卸载时关闭。

## 2. Profile 层（宿主启用）

在 profile 的 patch overlay 中加入：

```yaml
- insert:
    - id: browser-use
      name: '@deepseek-ai/dsh-browser-use'
    - id: browser-use-playwright
      name: '@deepseek-ai/dsh-experimental-browser-use-playwright-mcp'
      config:
        mode: launch
        headless: true
```

启动：

```sh
dsh --patch ./browser-use.yml --profile <profile> --port 3997 --no-open
```

- `mode: attach` + `endpoint` 可复用已存在的 Chromium；否则 `launch` 启动新的无头浏览器。
- 需要系统已安装 Chromium/Chrome；可用 `executablePath` 指定。
- 加载 provider 后创建的 Session 才拥有浏览器；已存在的 Session 不会自动接管。

## 3. Agent preset 层（工具可见）

在 agent preset 的 `cordis.yml` 中保持上述两行，并让系统提示词的工具排序把浏览器工具留给 `<unlisted-tools>`。**不要把浏览器工具名显式写进 `toolOrder`**：没有浏览器连接的 Session 会因提示词组装失败。

典型 preset：

```yaml
- name: '@deepseek-ai/dsh-system-prompt'
  config:
    toolOrder:
      - read
      - write
      # 浏览器工具不要列在这里，保持未列出
```

## 4. 验证

1. 启动后 `--dump-config` 应出现两行 browser-use。
2. 新建 Session，询问模型“打开 example.com 并读取标题”；工具调用应出现 Playwright MCP 的浏览器工具。
3. `mode: launch` 下退出 Session，浏览器进程应被回收；`attach` 下外部浏览器保留。
4. 卸载 provider 行后，新 Session 不再出现浏览器工具。

## 5. 限制

- 两个 provider 互斥，第二次注册会报错。
- MCP 初始化失败会拒绝 Session 创建/恢复；这是上游行为。
- Stagehand 的 AI 操作使用其自身配置的原生模型，不走 DSH 模型路由与计费。
- 浏览器登录态不进入 Session 日志；恢复 Session 得到新的浏览器状态。
- 需要网络访问与可用的 Chromium；当前环境已验证 provider 可安装，未做完整工具调用回归。
