# claw-mvp 从零开发检查清单

⚠️ 以下是根据 2026-05-26~27 实战经验总结的关键检查点。**开发 claw-mvp 时必须逐项核对**，确保不会踩坑。

---

## 阶段一：设计-审查

- [ ] Designer 输出到 `D:\openclaw\claw-mvp\01-设计文档.md`
- [ ] Developer 审查输出到 `D:\openclaw\claw-mvp\01-审查意见.md`
- [ ] 循环直到 Developer 输出 `✅ 审查通过`
- [ ] 最多 5 轮，超限向用户报告分歧

---

## 阶段二：编码（Developer 必须注意）

### 核心模块

- [ ] `src/llm.ts` — `Message` 接口必须包含 `reasoning_content?: string` 字段
- [ ] `src/llm.ts` — `LLMChunk` 类型必须包含 `reasoning_delta`
- [ ] `src/llm.ts` — `chat()` 方法签名包含 `tools` 参数
- [ ] `src/llm.ts` — 流处理：捕获 `delta.reasoning_content`，产出 `reasoning_delta` chunk
- [ ] `src/llm.ts` — `toOpenAIMessages()` 消息映射：assistant 消息有 `reasoning_content` 时必须附加
- [ ] `src/agent.ts` — `consumeStream()` 结构（⚠️ **最易出错**）：
  - ❌ **禁止** switch 语句后加 `return`（首 chunk 后退出）
  - ✅ `case "done"` 内部 return，循环结束后统一 return
- [ ] `src/agent.ts` — `consumeStream()` 返回的 `LLMFinalResult` 必须包含 `reasoning_content`
- [ ] `src/agent.ts` — 存储 assistant 消息时携带 `reasoning_content`（3 处）
- [ ] `src/agent.ts` — `Agent` 类接受 `AgentCallbacks` 参数（Web 模式用回调代替 stdout）
- [ ] `src/agent.ts` — `abortRequested` / `generating` 标志位
- [ ] `src/agent.ts` — `buildMessages()` 截断回溯逻辑（tool_calls/tool 配对保护）
- [ ] `src/agent.ts` — `run()` 调用 `chat()` 时传入 `this.tools.getDefinitions()`
- [ ] `src/agent.ts` — 搜索轮次限制：`searchRoundCount` 合并计数，由 `config.maxSearchRounds` 控制（默认 4）
- [ ] `src/agent.ts` — 反空话检测：`looksLikeChatter()` + `chatterRetryCount`（≤3 次）
- [ ] `src/agent.ts` — `maxHistoryTurns` 默认值 50，`maxIterations` 默认值 30
- [ ] `src/config.ts` — `ClawConfig` 接口包含 `maxIterations: number`（默认 30）和 `maxHistoryTurns: number`（默认 50）
- [ ] `src/config.ts` — System Prompt 从 `system-prompt.md` 加载，支持 `{workspace}`/`{platform}`/`{shell}` 占位符
- [ ] `src/config.ts` — 加载优先级: `config.json` > `system-prompt.md` > 内置回退
- [ ] `src/session.ts` — `SessionIndex` 格式 `{nameToId, ids}` + 向后兼容旧扁平格式
- [ ] `src/tools/index.ts` — `ToolRegistry` 接受 `ClawConfig`，模块化注册
- [ ] 所有内部 import 使用 `.ts` 后缀

### Web 模式

- [ ] `src/server.ts` — 原生 `node:http` 服务器，零框架依赖
- [ ] `src/server.ts` — SSE 推送 6 种事件（session/text/tool_call/tool_result/error/done）
- [ ] `src/server.ts` — REST API：POST /api/chat、GET/POST/DELETE /api/sessions、GET /api/config、GET/POST /api/models
- [ ] `src/server.ts` — 多会话管理：内存 Map，按 sessionId 隔离 Agent 实例
- [ ] `src/server.ts` — 静态文件服务 + MIME 类型映射
- [ ] `src/server.ts` — 端口解析：CLI `--port` > 环境变量 `CLAW_PORT` > 默认 3456
- [ ] `public/index.html` — 暗色主题 SPA 聊天界面（流式显示 + 工具调用可视化）

### System Prompt

- [ ] `system-prompt.md` — 独立文件，Markdown 格式，中文编写
- [ ] 包含安全策略（工作区沙箱、文件操作限制、读取限制、外部路径处理）
- [ ] 包含平台标识 `{platform}` `{shell}` + 搜索限制 + 反空话规则 + 任务聚焦约束
- [ ] 包含代码生成规则：Python 脚本禁用中文引号
- [ ] 长度控制在 600 字符以内（过长导致 LLM 行为退化）

### 工具模块

- [ ] `src/tools/exec.ts` — 危险命令黑名单按平台动态加载（Windows/Linux）
  - ⚠️ Windows 11 sudo 需加入黑名单
- [ ] `src/tools/exec.ts` — 编码修复：`encoding: "buffer"` + `TextDecoder('gbk')` 回退
- [ ] `src/tools/write.ts` — Windows 跨盘符拒绝
- [ ] `src/tools/read.ts` / `write.ts` — 路径解析去除多余 "workspace/" 前缀
- [ ] `src/tools/python.ts` — auto-pip bootstrap + 包名→模块名映射表（BUG-MVP-011）
- [ ] `src/tools/read.ts` — Office/PDF 格式检测 + Python 提取器（xlsx/docx/pptx/pdf）
- [ ] `src/tools/write.ts` — Office 格式写入：CSV→xlsx、md→docx、`# 标题`→pptx 自动转换
- [ ] `src/tools/download.ts` — 文件下载：流式写入（`pipeline` + `Readable.fromWeb`），SSRF 防护，文件名自动提取
- [ ] `src/tools/web_search.ts` — 三引擎并行（Bing + Google + 百度），`Promise.allSettled` + URL 去重
- [ ] `src/config.ts` — `proxy` 配置项，`loadConfig()` 时设置 `HTTPS_PROXY`/`HTTP_PROXY` 环境变量
- [ ] `src/config.ts` — `maxSearchRounds` 配置项（默认 4），替代硬编码常量

---

## 阶段二后：自测

- [ ] `node --experimental-strip-types --check src/agent.ts` 通过 **后，必须实际运行验证**
  - ⚠️ `--check` 可能通过但运行时仍然失败（brace 不匹配等）
- [ ] `node --experimental-strip-types src/index.ts` 能正常启动并响应
- [ ] 禁止使用 `Promise<"timeout">` 等函数内泛型（strip-types 不兼容）
- [ ] 禁止使用异步 IIFE + `Promise.race`（strip-types 大概率失败）
- [ ] **🔥 端到端冒烟测试**（BUG-MVP-001/002/003 教训，2026-05-27）：
  - 用简单任务验证完整工具调用链：如"读取 readme.txt 内容，写到 readme-copy.txt"
  - 确认 Agent 触发的是 `tool_calls`（OpenAI 格式），**不是** ````tool\nread(...)\n```` 文本
  - 确认多轮工具调用后 reasoning_content 正确传回（不报 400 错误）

---

## P0/P1 关键点（2026-05-27 实测发现）⚠️

以下是最容易遗漏的关键代码细节，必须在编码时逐项验证：

- [ ] **`src/llm.ts` — `chat()` 签名 + 传 tools**（BUG-MVP-001 P0）：签名含 `tools` 参数，传入 API
- [ ] **`src/agent.ts` — `run()` 传 tools**（BUG-MVP-003 P0）：调用 `chat()` 时传入 `this.tools.getDefinitions()`
- [ ] **`src/llm.ts` — `toOpenAIMessages()` 传回 reasoning**（BUG-MVP-002 P0）：assistant 消息附加 `reasoning_content`
- [ ] **路径去 workspace/ 前缀**（BUG-MVP-004 P1）：read/write 路径解析追加 `.replace(/^(\.?[\\/])?workspace[\\/]/, "")`
- [ ] **exec 编码 TextDecoder 修复**（BUG-MVP-005/007 P1）：`encoding: "buffer"` + `TextDecoder('gbk')` 回退，不用 chcp
- [ ] **maxHistoryTurns ≥ 50 + maxIterations = 30**（BUG-MVP-008 P1）：多工具场景消息增长快
- [ ] **session loadIndex 向后兼容**（BUG-MVP-009 P1）：检测 `parsed.nameToId` 是否存在
- [ ] **Windows 11 sudo 拦截**（BUG-MVP-010 P1）：Windows 黑名单加入 `/^sudo\s+/`
- [ ] **python auto-pip 包名映射表**（BUG-MVP-011 P1）：`python-docx`→`docx`，`python-pptx`→`pptx`

---

## P0/P1 关键点（2026-05-28 实战发现）⚠️

以下是在 2026-05-28 第二轮实战中发现的致命遗漏，必须在编码时逐项验证：

- [ ] **`src/agent.ts` — 默认 callbacks 输出**（BUG-MVP-012 P0）：构造函数无 callbacks 时必须设置 `onStream = process.stdout.write`
- [ ] **`src/tools/index.ts` — config 传递**（BUG-MVP-013 P0）：`register()` 必须调用 `module.setConfig?.(this.config)`
- [ ] **`src/agent.ts` — 反空话阈值 ≤ 30**（BUG-MVP-014 P0）：`looksLikeChatter()` 的 `CHATTER_MAX_LENGTH` 必须为 30
- [ ] **`src/index.ts` — 管道输入兼容**（BUG-MVP-015 P1）：readline `close` 需等待 `processing` 标志
- [ ] **Banner ASCII 字符**（BUG-MVP-016 P2）：不用 box-drawing 字符，使用 `=== ---` 等 ASCII
- [ ] **`public/index.html` — 严格参照模板**：以 `references/web/index.html` 为模板
- [ ] **`public/index.html` — SSE 用 `\n\n` 分割**（BUG-MVP-017 P0）：禁止 `lines.indexOf(line)` 取 data 行
- [ ] **`public/index.html` — `hasToolCalls` 复位**（BUG-MVP-018 P0）：创建 post-tool 气泡后立即 `hasToolCalls = false`

### Web 前端检查清单

- [ ] 深蓝紫主题：CSS 变量 `--bg: #1a1a2e`、`--accent: #4fc3f7`、`--accent2: #e040fb`
- [ ] 头部：模型下拉 + 会话下拉 + 按钮（新建/保存/清空）
- [ ] Welcome 初始屏（h2 + p）
- [ ] 消息区分：user（右对齐蓝色）+ assistant（左对齐深色边框）
- [ ] 可折叠工具步骤卡片：`.tool-step` + expand/collapse + 参数/结果详情
- [ ] 步骤状态：◎执行中 → ✓完成 / ✕失败(自动展开)
- [ ] steps-group 容器：左侧竖线
- [ ] 步骤 FIFO 队列：`toolStepsQueue` 按名称匹配 tool_call → tool_result
- [ ] 发送按钮：生成中变红色"停止"
- [ ] Toast 通知
- [ ] 客户端断开安全：SSE send 用 try/catch 包裹
- [ ] `server.ts` API 对齐：`/api/sessions` GET 合并活跃+持久化、`/api/sessions/load` 接受 name 参数、`/api/sessions/:id/save` 接受请求体 name

---

## 阶段三：测试

- [ ] **项目 README.md** — 基于实际代码方案生成 `D:\openclaw\claw-mvp\README.md`
  - 内容以实际代码为准，仅参考 [README.template.md](README.template.md) 的章节结构
  - 不得照搬模板内容，版本号/依赖/配置/功能等必须与实现一致
- [ ] Tester 测试报告输出到 `D:\openclaw\claw-mvp\03-测试报告.md`
- [ ] 覆盖所有工具（7个）、配置模块、会话模块、Web 服务
- [ ] CLI 模式端到端冒烟测试
- [ ] Web 模式端到端测试（SSE 流式 + API）
- [ ] 循环直到 Tester 输出 `✅ 测试全部通过`
- [ ] 最多 10 轮

---

## 阶段三后：端到端验证

- [ ] 确保 `~/.claw/config.json` 中 API Key **不含 Unicode 省略号**（U+2026，非法 HTTP Header）
- [ ] CLI 模式启动：`node --experimental-strip-types src/index.ts` 正常进入 REPL
- [ ] Web 模式启动：`node --experimental-strip-types src/server.ts` 正常监听端口
- [ ] 使用真实任务测试（如"帮我总结 agent 技术发展，以 PPT 形式给我"）
- [ ] 验证：搜索 ≤4 轮被拦截 → Agent 调用 write 生成文件 → exec 执行 → 有错误时 read + edit 修复
- [ ] 验证：Agent 不会陷入"我来做"→空响应循环（反空话机制生效）
- [ ] 验证 Web 界面：浏览器打开正常，SSE 流式显示正常，多会话切换正常
- [ ] 问题解决后清理所有 `[DEBUG]`、`[R]`、`[T]`、`process.stderr.write` 调试日志

---

## 🔒 安全必须检查项（2026-05-27 新增）

- [ ] **read 工具 workspace 边界检查**：拒绝绝对路径、UNC 路径、路径遍历（SEC-001）
  - 检查函数必须在 `existsSync`/`readFileSync` 等文件系统调用之前执行
  - Windows 需跨盘符检测
- [ ] **exec 工具 PowerShell 编码命令拦截**：`-EncodedCommand`/`-Enc`/`-ec` 参数（SEC-002）
- [ ] **exec 工具下载执行链拦截**：`iex (iwr ...)`、`curl ... | bash`（SEC-003）
- [ ] **exec 工具环境变量泄露拦截**：`Get-ChildItem env:`、`ls env:`（SEC-006）
- [ ] **exec 工具 Windows 管理命令拦截**：`schtasks`、`netsh`、`net user`（SEC-009/011）
- [ ] **exec 工具权限修改命令拦截**：`icacls`、`cacls`、`takeown`、`reg add`
- [ ] **web_fetch SSRF 多层防御**（SEC-027~030）：
  - IPv6 mapped IPv4 检测
  - 十进制/十六进制 IP 检测
  - 云元数据端点 (169.254.0.0/16) 拦截
  - `file://` 协议拒绝
- [ ] **python 工具代码安全检查**（SEC-042）：
  - 拦截 `os.system`、`subprocess`、`eval`/`exec`、`ctypes`、网络模块
  - 拦截 `__import__('os')` 动态导入
- [ ] **python 工具 auto-pip bootstrap 包名映射**（BUG-MVP-011 🔥）：
  - pip 包名 ≠ Python import 模块名！必须有 `_pkg_to_mod` 映射表
  - 关键映射：`python-docx`→`docx`，`python-pptx`→`pptx`
  - 不能依赖 `replace("-", "_")` 启发式转换
  - 新增 auto-pip 依赖前，先验证 `pip install <包名>` 后 `python -c "import <模块名>"` 能成功
- [ ] **UNC 路径优先拦截**：在任何文件操作前检测 `\\\\` 或 `//` 开头并拒绝（SEC-019）
- [ ] **exec 文件操作沙箱**（SEC-010 🔴 实战发现）：
  - 检测 `Copy-Item`/`Move-Item` 的 `-Destination` 参数
  - 检测 `Out-File`/`Export-*` 的 `-FilePath` 参数
  - 检测 `Set-Content`/`Add-Content` 的 `-Path` 参数
  - 检测 cmd `copy`/`move`/`xcopy`/`robocopy` 目标
  - 检测 shell 重定向 `>` `>>` 目标路径
  - 展开环境变量后校验目标是否在 workspace 内
  - ⚠️ 这是 write 沙箱被 exec 绕过的关键防线
- [ ] **🔒 System Prompt 安全策略**（SEC-011 🔴 最关键的防线）：
  - ⚠️ **规则匹配不可靠，安全约束必须写入 System Prompt！**
  - 明确 workspace 是唯一可操作目录（LLM 语义理解 > 正则规则匹配）
  - 禁止 LLM 使用 exec 进行任何文件复制/移动/传输操作
  - 禁止读取/写入 workspace 外的路径
  - 遇到外部路径请求时，说明限制并提供替代方案
  - **设计原则**: 策略表达 > 规则匹配。exec 正则检查保留为纵深防御（defense in depth）

---

## 迭代测试方法论（2026-05-27 新增）

- [ ] 每次迭代前清理工作区残留文件（避免 "created"→"overwritten" 误判）
- [ ] 网络依赖功能（web_search/web_fetch）可能暂时不可用，测试应接受 "unavailable" 作为合理的降级结果
- [ ] 用独立 test-harness 脚本逐模块测试，不依赖 LLM
- [ ] 测试覆盖：工具函数输入输出、错误处理、安全拦截、编码兼容
- [ ] 每个 bug 修复后必须重新运行全部 25 项测试通过
- [ ] 迭代终止条件：所有测试通过 + 无新增代码 bug

---

## 功能测试用例集

> 以下测试用例用于功能迭代时逐项验证。每个工具/模块至少一个测试用例。

### read 工具

- [ ] **TC-READ-001: 读取纯文本文件** — workspace 中创建 test.txt，内容为 "hello world"，read 返回 content 正确。通过: success=true, content="hello world"
- [ ] **TC-READ-002: 读取不存在文件** — 读取 "missing.txt"。通过: success=false, error 包含 "不存在"
- [ ] **TC-READ-003: offset/limit 分页** — 从第 2 行开始读 1 行。通过: content 只包含 "line 2"
- [ ] **TC-READ-004: 绝对路径拒绝** — 读取 "C:\\Windows\\System32\\drivers\\etc\\hosts"。通过: success=false
- [ ] **TC-READ-005: UNC 路径拒绝** — 读取 "\\\\localhost\\C$\\test"。通过: success=false
- [ ] **TC-READ-006: 路径遍历拒绝** — 读取 "..\\..\\..\\Windows\\System32"。通过: success=false
- [ ] **TC-READ-007: workspace/ 前缀去除** — 路径含 "workspace/" 前缀时自动去除。通过: success=true

### write 工具

- [ ] **TC-WRITE-001: 创建纯文本文件** — 写入 "test_write.txt"，内容 "hello write"。通过: 文件存在且内容正确, action="created"
- [ ] **TC-WRITE-002: 覆盖已有文件** — 再次写入同一文件。通过: action="overwritten"，新内容生效
- [ ] **TC-WRITE-003: 追加模式** — append=true 追加内容。通过: action="appended"，内容为原有+新增
- [ ] **TC-WRITE-004: UNC 路径拒绝** — 写入 UNC 路径。通过: success=false
- [ ] **TC-WRITE-005: 缺少参数** — 缺少 path 或 content。通过: success=false, error 清晰

### edit 工具

- [ ] **TC-EDIT-001: 精确编辑替换** — "Draft"→"Final"。通过: success=true，替换生效
- [ ] **TC-EDIT-002: 0 次匹配报错** — oldText 不在文件中。通过: success=false, 提示"未找到"
- [ ] **TC-EDIT-003: 多次匹配报错** — oldText 出现多次。通过: success=false, 提示匹配数和行号
- [ ] **TC-EDIT-004: 路径安全检查** — 遍历路径或绝对路径。通过: success=false

### exec 工具

- [ ] **TC-EXEC-001: 基本命令执行** — `echo hello`。通过: success=true, stdout 包含输出
- [ ] **TC-EXEC-002: 危险命令拦截(sudo)** — `sudo rm -rf /`。通过: success=false, 含"拦截"
- [ ] **TC-EXEC-003: 危险命令拦截(format)** — `format C:`。通过: success=false
- [ ] **TC-EXEC-004: PowerShell 编码命令拦截** — `-EncodedCommand`。通过: success=false
- [ ] **TC-EXEC-005: 下载执行链拦截** — `iex (iwr ...)`。通过: success=false
- [ ] **TC-EXEC-006: 环境变量枚举拦截** — `Get-ChildItem env:`。通过: success=false
- [ ] **TC-EXEC-007: 管理命令拦截** — `net user test /add`。通过: success=false
- [ ] **TC-EXEC-008: 中文编码** — `echo 测试中文`。通过: stdout 无乱码

### session 模块

- [ ] **TC-SESSION-001: 保存/加载** — 保存含消息的会话，再加载。通过: 加载后消息一致
- [ ] **TC-SESSION-002: 列出会话** — 保存后 list 能查到。通过: 列表含已保存会话
- [ ] **TC-SESSION-003: 删除会话** — 删除后 load 返回 null。通过: 删除成功

### python 工具

- [ ] **TC-PYTHON-001: 基本脚本** — `print('hello')`。通过: stdout 含 "hello"
- [ ] **TC-PYTHON-002: 中文输出** — `print('你好')`。通过: 无乱码
- [ ] **TC-PYTHON-003: os.system 拦截** — 含 `os.system`。通过: success=false
- [ ] **TC-PYTHON-004: subprocess 拦截** — 含 `subprocess.run`。通过: success=false
- [ ] **TC-PYTHON-005: auto-pip** — 脚本 import openpyxl。通过: 自动安装并执行成功

### CLI 模式

- [ ] **TC-CLI-001: 启动 + 斜杠命令** — 启动 CLI，验证 Banner（ASCII 无乱码），/help 显示 10 条命令，/history 显示无历史。通过: 全部正常
- [ ] **TC-CLI-002: 管道输入** — `echo /exit | npm start`。通过: 正常退出，不卡住

### Web 模式

- [ ] **TC-WEB-001: 服务启动 + 静态文件** — 启动 Web 服务，浏览器访问首页。通过: Welcome 屏显示，模型下拉有选项
- [ ] **TC-WEB-002: API 端点** — GET /api/config, /api/models, POST /api/sessions。通过: 返回正确 JSON
- [ ] **TC-WEB-003: SSE 聊天** — POST /api/chat，监听 SSE 事件。通过: 收到 session + text + done 事件

### Agent 核心（需 LLM）

- [ ] **TC-AGENT-001: 多轮工具调用** — 读取文件再写入另一个文件。通过: 目标文件存在且内容正确
- [ ] **TC-AGENT-002: 反空话检测** — 纯"我来做"类回复被拦截重试。通过: 不会死循环
- [ ] **TC-AGENT-003: 搜索限制** — 4 轮搜索后继续调搜索被拦截。通过: 搜索预算耗尽后综合回答
