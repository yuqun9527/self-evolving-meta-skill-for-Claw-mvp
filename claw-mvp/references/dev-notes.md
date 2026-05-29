# claw-mvp 开发注意事项

在修改本项目代码时，必须注意以下事项（详见 [bug-tracker.md](bug-tracker.md)）：

## 语法与工具链

1. **导入后缀**：内部 import 必须用 `.ts` 后缀（不能是 `.js`），否则 Node.js strip-types 无法解析
2. **`--experimental-strip-types` 陷阱**：`--check` 通过 ≠ 实际可运行；函数内不能用 `type` 别名；异步 IIFE + `Promise.race` 大概率失败
3. **package.json**：只有 `openai` 一个运行时依赖，无 `typescript`/`tsx` 等 devDependency，不设 `build`/`dev` 脚本

## Agent 核心

4. **consumeStream 结构**：`switch` 语句后**不能在 for-await 内加 `return`**，否则首 chunk 后立即退出（BUG-006）。代码模式见 [code-patterns.md](code-patterns.md)
5. **DeepSeek V4 推理**：`reasoning_content` 必须在消息映射和存储时携带并传回，否则 API 400（BUG-007）。见 [code-patterns.md](code-patterns.md)
6. **Agent Callbacks 模式**：`Agent` 构造函数接受 `AgentCallbacks` 参数（`onStream`/`onToolCall`/`onToolResult`/`onError`），CLI 模式用默认 stdout 输出，Web 模式通过回调注入 SSE 推送
7. **消息截断保护**：`buildMessages()` 截断时必须保证 tool_calls/tool 消息配对完整，不能切断 assistant(tool_calls) 与其后续 tool 消息
8. **搜索预算合并计算**：`web_search` 和 `web_fetch` 共享轮次上限，统一用 `searchRoundCount` 计数器，上限由 `config.maxSearchRounds` 配置（默认 4）

## 文件操作

9. **write 跨盘符**：Windows 下跨盘符写文件需特殊处理
10. **buildMessages 截断**：截断历史消息时不能切断 tool_calls/tool 配对，需回溯检查

## 安全

11. **exec 危险命令检测**：`isDangerous()` 对命令做了 `toLowerCase()`，注意正则大小写

## LLM 行为约束

12. **System Prompt 长度**：控制在 600 字符以内，过长导致 LLM 行为退化（只回复"我来"后空响应）
13. **Python 中文引号**：LLM 生成的 Python 脚本中，中文引号 `""` `''` 会与 Python 字符串引号冲突。System Prompt 需明确禁止，或用三引号包裹
14. **API Key 特殊字符**：Unicode 省略号（U+2026）不是合法 HTTP Header 值，不要用省略号占位

## 配置与架构

15. **System Prompt 独立配置文件**：不要硬编码在 `config.ts` 中。应放在项目根目录的 `system-prompt.md`（Markdown 格式，中文编写），由 `config.ts` 按优先级加载：用户 `config.json` > `system-prompt.md` > 内置回退。安全策略用自然语言写入 System Prompt（LLM 语义理解 > 工具层正则匹配）
16. **config 导出**：只导出有限 API，注意可用接口

## Agent 输出与交互

17. **Agent 默认输出**：Agent 构造函数必须在无 callbacks 时设置默认 `onStream = process.stdout.write`，否则 CLI 模式无任何输出（BUG-MVP-012）。同时设置默认的 `onToolCall`/`onToolResult`/`onError` 以便 CLI 模式显示工具执行进度
18. **反空话阈值**：`looksLikeChatter()` 阈值应 ≤ **30 字符**，检测模式限定为纯计划性短句（如 `^(我来|我马上|let me )`），避免误伤正常回复（BUG-MVP-014）
19. **管道输入处理**：CLI 的 readline `close` 事件不能直接 `process.exit(0)`，需检查 `processing` 标志，异步等待 Agent 完成后再退出（BUG-MVP-015）
20. **Banner 兼容性**：Banner 必须使用 ASCII 字符（`===` 分割线），禁止使用 box-drawing 字符（╔══），确保 Windows PowerShell GBK 终端正常显示（BUG-MVP-016）

## 工具配置传递

21. **ToolRegistry 必须传递 config**：`ToolRegistry.register()` 中必须调用 `module.setConfig?.(this.config)`，将 workspace 等配置注入每个工具模块（BUG-MVP-013）。各工具模块（read/write/edit/exec/download/python）的 `getWorkspace()` 依赖此配置
22. **新增工具模块模板**：每个需要 workspace 的工具模块必须包含 `let _config: ClawConfig | null = null` + `export function setConfig(c: ClawConfig)` + `function getWorkspace(): string` 三段式
23. **download 工具流式写入**：下载大文件时必须使用 `pipeline(Readable.fromWeb(resp.body), createWriteStream(path))`，避免内存溢出。超时设为 60s。文件名自动从 Content-Disposition 头部或 URL 路径提取，重名自动追加 `_(1)` `_(2)`
24. **proxy 配置注入**：`config.ts` 的 `loadConfig()` 中通过 `process.env.HTTPS_PROXY = proxy` 设置代理，使 Node.js 原生 `fetch` 自动走代理。不需要在 `web_search` / `web_fetch` / `download` 中手动处理代理，环境变量已全局生效

## Web 前端（2026-05-28 新增）

25. **Web 界面模板**：⚠️ **后续所有 Web 界面必须以 `references/web/index.html`（技能自带模板）为严格参考模板**，确保：
    - 深蓝紫主题（CSS 变量 `--bg: #1a1a2e` 系列）
    - 头部：模型下拉 + 会话下拉 + 操作按钮（新建/保存/清空）
    - 可折叠工具步骤卡片（`.tool-step` → expand/collapse）
    - 步骤 FIFO 队列匹配 `tool_call` → `tool_result`
    - 发送按钮：生成中变为红色"停止"
    - Toast 通知
    - 不得自行设计新的视觉风格或交互模式
26. **SSE 解析规范**：前端必须用 `\n\n` 双换行分割完整事件块，逐块独立解析 `event:` 和 `data:` 行；**禁止**使用 `indexOf` 偏移取值——同一 buffer 内多事件时必然串位（BUG-MVP-017）
27. **hasToolCalls 一次性标志**：前端 `hasToolCalls` 在工具调用后收到第一个 text 事件时创建新消息气泡，之后**必须立即复位** `hasToolCalls = false`，否则后续每个 text chunk 都创建新气泡（BUG-MVP-018）
28. **消息气泡生命周期**：每次 `sendMessage()` 调用创建一个 assistant 消息气泡 + 一个 steps-group 容器；`finishGeneration()` 时清理引用；多轮 tool-call 迭代中，每轮 post-tool 文本创建独立气泡（通过 `hasToolCalls` 触发）
29. **API 对齐**：前端依赖的 API 端点（`/api/models`、`/api/sessions`、`/api/sessions/load`、`/api/sessions/:id/save`、`/api/chat` model 参数）必须在 `server.ts` 中正确实现并返回预期格式

## 测试与运维（2026-05-29 新增）

30. **🛑 禁止 `taskkill /f /im node.exe` 全局杀进程**：claw-mvp Web 服务器在测试完成后需要停止时，**绝对不能**使用 `taskkill /f /im node.exe`——这会无差别杀掉系统上所有 node.exe 进程，包括正在运行该测试的 OpenClaw 自身。正确做法：
    - 先 `netstat -ano | findstr :<端口>` 找到监听端口的 PID
    - 再 `taskkill /pid <PID>` 精确终止
    - 或用 `taskkill /fi "WINDOWTITLE eq claw-mvp*"` 按窗口标题过滤
    - Linux/macOS: `lsof -ti :<端口> | xargs kill`
