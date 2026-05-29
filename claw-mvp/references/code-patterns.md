# claw-mvp 代码模式参考

## 工具实现模板

添加新工具的标准模板：

```typescript
// src/tools/newtool.ts

export const definition = {
  type: "function" as const,
  function: {
    name: "newtool",
    description: "简短描述工具功能",
    parameters: {
      type: "object" as const,
      properties: {
        arg1: {
          type: "string",
          description: "参数 1 说明",
        },
      },
      required: ["arg1"],
    },
  },
};

export interface NewToolResult {
  success: boolean;
  // ... 其他字段
  error?: string;
}

export async function execute(args: Record<string, unknown>): Promise<NewToolResult> {
  const arg1 = args["arg1"] as string;
  if (!arg1) {
    return { success: false, error: "Missing required parameter: arg1" };
  }
  // ... 实现
  return { success: true };
}
```

然后在 `src/tools/index.ts` 注册：

```typescript
import * as newTool from "./newtool.ts";

export class ToolRegistry {
  constructor(config: ClawConfig) {
    this.config = config;
    // ... 其他工具
    this.register(newTool);
  }

  // 模块化注册：auto-extract definition.function.name
  private register(module: { definition: typeof readTool.definition; execute: ToolExecuteFn }): void {
    const name = module.definition.function.name;
    this.tools.set(name, { definition: module.definition, execute: module.execute });
  }
}
```

## Agent Loop 完整版（含搜索限制 + 反空话 + 回调）

```typescript
// agent.ts - Agent.run() 完整主循环

import type { Message, ToolCall, LLMChunk, LLMFinalResult } from "./llm.ts";

export interface AgentCallbacks {
  onStream?: (text: string) => void;
  onToolCall?: (name: string, args: Record<string, unknown>) => void;
  onToolResult?: (name: string, result: unknown) => void;
  onError?: (msg: string) => void;
}

const SEARCH_TOOLS = new Set(["web_search", "web_fetch"]);
const CHATTER_PATTERNS = [
  /^[我I].{0,5}(来|开始|马上|准备|立即|编写|创建|生成|现在)/,
  /^(let me|I will|I'll|going to|let's|now)\b/im,
  /我是.{0,20}(claw|AI|助手).{30,}/,
  /(工作区|workspace).{0,30}(如下|列表|包含|文件|目录).{0,30}(:|-|—)/,
];

export class Agent {
  private messages: Message[] = [];
  private config: ClawConfig;
  private tools: ToolRegistry;
  private llmClient: LLMClient;
  private searchRoundCount = 0;
  private chatterRetryCount = 0;
  private callbacks: AgentCallbacks;
  public abortRequested = false;
  public generating = false;

  constructor(config: ClawConfig, callbacks?: AgentCallbacks) { ... }

  async run(userInput: string): Promise<void> {
    this.messages.push({ role: "user", content: userInput });
    let iteration = 0;
    this.abortRequested = false;

    while (iteration < this.config.maxIterations && !this.abortRequested) {
      iteration++;
      this.generating = true;

      // BUG-MVP-001/003: 传入 tools 定义
      const generator = this.llmClient.chat(
        this.buildMessages(),
        undefined,  // signal
        this.tools.getDefinitions()
      );

      const result = await this.consumeStream(generator);
      this.generating = false;

      if (this.abortRequested) break;

      // 文本响应 → 反空话检测
      if (result.type === "text") {
        const textContent = result.content.trim();
        if (textContent.length === 0 || this.looksLikeChatter(textContent)) {
          this.chatterRetryCount++;
          if (this.chatterRetryCount > 3) {
            this.messages.push({ role: "assistant", content: textContent || "(no response)", reasoning_content: result.reasoning_content });
            break;
          }
          this.messages.push({ role: "user", content: "[SYSTEM] 你的上一条回复是空话。请直接简洁回应用户，1-3句话。" });
          continue;
        }
        this.chatterRetryCount = 0;
        this.messages.push({ role: "assistant", content: textContent, reasoning_content: result.reasoning_content });
        break;
      }

      // tool_calls → 搜索限制 + 执行
      if (result.type === "tool_calls") {
        this.chatterRetryCount = 0;
        const toolCalls = [...result.toolCalls];
        const searchTCs = toolCalls.filter(tc => SEARCH_TOOLS.has(tc.function.name));
        if (searchTCs.length > 0) this.searchRoundCount++;

        // 搜索超限
        if (this.searchRoundCount > this.config.maxSearchRounds && toolCalls.some(tc => SEARCH_TOOLS.has(tc.function.name))) {
          this.messages.push({ role: "assistant", content: null, tool_calls: [...toolCalls], reasoning_content: result.reasoning_content });
          for (const tc of toolCalls) {
            if (SEARCH_TOOLS.has(tc.function.name)) {
              this.messages.push({ role: "tool", tool_call_id: tc.id, content: JSON.stringify({ success: false, error: `搜索预算已耗尽(${this.config.maxSearchRounds}轮上限)` }) });
            } else {
              // 非搜索工具正常执行
            }
          }
          continue;
        }

        // BUG-007: 存储 assistant 消息时携带 reasoning_content
        this.messages.push({ role: "assistant", content: null, tool_calls: [...toolCalls], reasoning_content: result.reasoning_content });

        // 执行所有工具
        for (const tc of toolCalls) {
          const tcArgs = JSON.parse(tc.function.arguments);
          this.callbacks.onToolCall?.(tc.function.name, tcArgs);
          const execResult = await this.tools.execute(tc.function.name, tcArgs);
          this.callbacks.onToolResult?.(tc.function.name, execResult);
          this.messages.push({ role: "tool", tool_call_id: tc.id, content: JSON.stringify(execResult) });
        }
      }
    }

    if (iteration >= this.config.maxIterations) {
      console.log("\n⚠ 达到最大迭代次数");
    }
  }
}
```

## Session 持久化模式

```typescript
// session.ts

export class SessionManager {
  private sessionsDir: string;   // ~/.claw/sessions/
  private indexFile: string;     // ~/.claw/sessions/index.json

  async save(name?, model?, messages?): Promise<string> {
    // 1. 读取索引（name → uuid 映射）
    // 2. 如果有同名会话，复用 uuid
    // 3. 否则生成新 uuid
    // 4. 写 <uuid>.json 文件
    // 5. 更新索引
  }

  async load(idOrName: string): Promise<SessionData> {
    // 1. 先查索引找 uuid
    // 2. 读 <uuid>.json
    // 3. 返回 SessionData
  }
}
```

## 流式输出处理

```typescript
// llm.ts - LLMClient.chat() 

async *chat(params: ChatParams): AsyncGenerator<LLMChunk> {
  const response = await this.client.chat.completions.create({
    model: this.model,
    messages: params.messages,
    tools: params.tools,
    stream: true,
  });

  for await (const chunk of response) {
    const delta = chunk.choices?.[0]?.delta as Record<string, unknown>;
    
    // DeepSeek V4: 捕获 reasoning_content
    const rc = delta.reasoning_content ?? (chunk.choices?.[0] as any).reasoning_content;
    if (typeof rc === "string" && rc) {
      yield { type: "reasoning_delta", delta: rc };
    }
    
    if (delta.content) {
      yield { type: "text_delta", delta: delta.content as string };
    }
    
    if (delta.tool_calls) {
      yield { type: "tool_call_delta", index, delta };
    }
    
    if (chunk.choices?.[0]?.finish_reason) {
      yield { type: "done", finish_reason };
      return;
    }
  }
}
```

## Agent consumeStream 正确模式

⚠️ **陷阱**：switch 语句后不能在 for-await 内加 return，否则首 chunk 后就退出循环，导致 textContent 始终为空。

```typescript
// agent.ts - Agent.consumeStream() ✅ 正确

private async consumeStream(generator): Promise<LLMFinalResult> {
  let textContent = "";
  let reasoningContent = "";

  for await (const chunk of generator) {
    if (this.abortRequested) break;

    switch (chunk.type) {
      case "reasoning_delta":
        reasoningContent += chunk.delta;
        break;
      case "text_delta":
        textContent += chunk.delta;
        process.stdout.write(chunk.delta);
        break;
      case "tool_call_delta":
        // ... 累积 tool call
        break;
      case "done":
        if (chunk.finish_reason === "tool_calls" && toolCallAcc.size > 0) {
          return { type: "tool_calls", toolCalls, reasoning_content: reasoningContent };
        }
        return { type: "text", content: textContent, reasoning_content: reasoningContent };
    }
    // ❌ 不要在这里加 return！
  }

  return { type: "text", content: textContent, reasoning_content: reasoningContent };
}
```

## DeepSeek V4 推理模型适配

```typescript
// Message 类型增加 reasoning_content 字段
export interface Message {
  role: "system" | "user" | "assistant" | "tool";
  content: string | null;
  tool_calls?: ToolCall[];
  tool_call_id?: string;
  reasoning_content?: string;  // DeepSeek V4 thinking 模式
}

// llm.ts: 消息映射时传回 reasoning_content
if (m.role === "assistant" && m.reasoning_content) {
  (msg as Record<string, unknown>).reasoning_content = m.reasoning_content;
}

// agent.ts: 存储 assistant 消息时携带 reasoning_content
this.messages.push({
  role: "assistant",
  content: null,
  tool_calls: toolCalls,
  reasoning_content: result.reasoning_content,  // 必须传回！
});
```

⚠️ **注意**：`choice.delta.reasoning_content` 可能被 OpenAI SDK 的类型定义隐藏，
需双重 fallback：`(delta as Record<string, unknown>).reasoning_content ?? (choice as any).reasoning_content`

## 反空话检测

```typescript
// agent.ts - 防止 LLM 用"我来做"等空话消耗迭代

if (result.type === "text") {
  const textContent = result.content.trim();

  // 空响应或短"计划性表述" → 强制重试（最多 3 次）
  if (textContent.length === 0 ||
      (textContent.length < 300 && looksLikeChatter(textContent))) {
    if (chatterRetryCount++ > 3) {
      // 放弃，避免死循环
      this.messages.push({ role: "assistant", content: textContent || "(no response)" });
      break;
    }
    this.messages.push({
      role: "user",
      content: "[SYSTEM] Your last response was empty or just planning talk. Use tools NOW.",
    });
    continue;
  }

  chatterRetryCount = 0;  // 恢复正常
  this.messages.push({ role: "assistant", content: textContent, reasoning_content: result.reasoning_content });
  break;
}

// 检测空话的启发式规则
private looksLikeChatter(text: string): boolean {
  const patterns = [
    /开始|马上|准备|立即|编写|创建|生成|现在/,
    /let me|I will|I'll|going to|let's|now|start|begin|create|prepare/i,
  ];
  return patterns.some((p) => p.test(text));
}
```

## 搜索轮次限制（可配置）

```typescript
// agent.ts - 搜索轮次上限由 config.maxSearchRounds 控制（默认 4）

// 在工具执行前检查
if (searchToolsInRound.length > 0) {
  this.searchRoundCount++;
  if (this.searchRoundCount > this.config.maxSearchRounds) {
    // BUG-005 修复：先 push assistant(tool_calls)，再 push tool error result
    const originalToolCalls = [...toolCalls];
    const nonSearch = toolCalls.filter(tc => !isSearchTool(tc));
    toolCalls.length = 0;
    toolCalls.push(...nonSearch);

    this.messages.push({ role: "assistant", content: null, tool_calls: originalToolCalls, reasoning_content });
    for (const tc of searchToolsInRound) {
      this.messages.push({ role: "tool", tool_call_id: tc.id, content: JSON.stringify({
        success: false,
        error: "Search budget exhausted. Synthesize now."
      })});
    }
    if (toolCalls.length === 0) continue;  // 全是搜索工具，下一轮
  }
}
```

## Ctrl+C 中断处理

```typescript
// index.ts

process.on("SIGINT", () => {
  if (agent.generating) {
    agent.abortRequested = true;   // 中断 LLM 流
  } else {
    ctrlCCount++;
    if (ctrlCCount >= 2) process.exit(0);  // 双击退出
  }
});
```

## Web 服务模式 (server.ts) 🆕

### SSE 推送模式

```typescript
// server.ts - SSE 辅助函数

import { ServerResponse } from "node:http";

function sseInit(res: ServerResponse): void {
  res.writeHead(200, {
    "Content-Type": "text/event-stream; charset=utf-8",
    "Cache-Control": "no-cache",
    Connection: "keep-alive",
    "X-Accel-Buffering": "no",  // 禁用 nginx 缓冲
  });
}

function sseSend(res: ServerResponse, event: string, data: unknown): void {
  res.write(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`);
}
```

### 聊天处理（SSE 流式 + Agent Callbacks）

```typescript
// server.ts - handleChat()

async function handleChat(req, res) {
  const body = await readBody(req);
  const { message, sessionId } = JSON.parse(body);
  const sid = sessionId || "default";
  sseInit(res);

  const sa = getOrCreateSession(sid);
  sseSend(res, "session", { sessionId: sid, model: sa.config.model, workspace: sa.config.workspace });

  // 并发保护：如果正在生成，先中断
  if (sa.agent.generating) {
    sa.agent.abortRequested = true;
    await new Promise(r => setTimeout(r, 300));
  }

  // 注入 Web 回调（SSE 推送代替 stdout）
  (sa.agent as any).callbacks = {
    onStream(text) { sseSend(res, "text", { content: text }); },
    onToolCall(name, args) { sseSend(res, "tool_call", { name, args }); },
    onToolResult(name, result) { sseSend(res, "tool_result", { name, result }); },
    onError(msg) { sseSend(res, "error", { message: msg }); },
  };

  await sa.agent.run(message.trim());
  sseSend(res, "done", { sessionId: sid });

  if (!res.writableEnded) res.end();
}
```

### 多会话管理

```typescript
// server.ts - 会话管理

interface SessionAgent {
  agent: Agent;
  config: ClawConfig;
  createdAt: number;
}

const sessions = new Map<string, SessionAgent>();

function getOrCreateSession(sessionId: string): SessionAgent {
  if (!sessions.has(sessionId)) {
    const config = loadConfig();
    const agent = new Agent(config);
    sessions.set(sessionId, { agent, config, createdAt: Date.now() });
  }
  return sessions.get(sessionId)!;
}
```

### 静态文件服务

```typescript
// server.ts - serveStatic()

const MIME_TYPES: Record<string, string> = {
  ".html": "text/html; charset=utf-8",
  ".css": "text/css; charset=utf-8",
  ".js": "application/javascript; charset=utf-8",
  ".json": "application/json; charset=utf-8",
  ".png": "image/png",
  ".svg": "image/svg+xml",
  ".ico": "image/x-icon",
};

function serveStatic(res: ServerResponse, urlPath: string): void {
  if (urlPath === "/" || urlPath === "") urlPath = "/index.html";
  const filePath = join(PUBLIC_DIR, urlPath);
  if (!existsSync(filePath)) {
    res.writeHead(404);
    res.end("404 Not Found");
    return;
  }
  const ext = extname(filePath).toLowerCase();
  const contentType = MIME_TYPES[ext] || "application/octet-stream";
  res.writeHead(200, { "Content-Type": contentType });
  res.end(readFileSync(filePath));
}
```

### 端口解析（三级优先级）

```typescript
// server.ts - resolvePort()

function resolvePort(): number {
  const args = process.argv.slice(2);
  for (let i = 0; i < args.length; i++) {
    if ((args[i] === "--port" || args[i] === "-p") && args[i + 1]) {
      return parseInt(args[i + 1], 10);
    }
  }
  for (const arg of args) {
    if (/^\d+$/.test(arg)) return parseInt(arg, 10);
  }
  return parseInt(process.env.CLAW_PORT || "3456", 10);
}
```

## 斜杠命令路由模式

```typescript
// index.ts - 斜杠命令路由

async function handleSlashCommand(cmd, agent, sessionManager, rl) {
  const [command, arg] = cmd.split(/\s+/);
  
  switch (command) {
    case "/file":   // 注入文件到上下文
    case "/model":  // 切换模型
    case "/clear":  // 清屏+清空历史
    case "/history":// 显示消息历史摘要
    case "/save":   // 保存会话
    case "/load":   // 加载会话
    case "/list":   // 列出所有已保存会话
    case "/delete": // 删除会话
    case "/exit":   // 退出
    case "/help":   // 显示帮助
  }
}
```

添加新斜杠命令：在 switch 中添加 case，并更新 `HELP_TEXT` 和 `completer` 数组。

## Python 工具模式 (v0.2.0 🆕)

### auto-pip bootstrap 模式

```typescript
// python.ts - pipBootstrap()
// 在用户脚本前插入依赖检查和安装代码

function buildBootstrap(packages: string[]): string {
  const pkgList = JSON.stringify(packages);
  return `
import subprocess, sys, importlib
_auto_packages = ${pkgList}
for _pkg in _auto_packages:
    _mod = _pkg.replace("-", "_")
    try:
        importlib.import_module(_mod)
    except ImportError:
        subprocess.check_call(
            [sys.executable, "-m", "pip", "install", _pkg, "-q", "-q"],
            stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL
        )
# ---- end bootstrap ----
`;
}
```

⚠️ **重要**：AUTO_PACKAGES 只包含轻量级核心包（openpyxl/python-docx/python-pptx/pdfplumber/PyPDF2）。
重型包（pandas/matplotlib）必须通过 `packages` 参数按需安装，否则 60s 超时不够用。

### Python 工具执行流程

```
用户脚本 → write temp .py file → exec python → capture stdout/stderr → cleanup temp
                ↑
        bootstrap (auto-pip)
```

- 临时文件命名：`.claw-python-{uuid}.py`，写入 workspace
- 执行环境：`PYTHONIOENCODING=utf-8` + `PYTHONUTF8=1`（解决 Windows GBK）
- 超时：默认 60s，最大 300s
- 清理：无论成功失败都删除临时文件

## Office/PDF 格式检测与处理模式 (v0.2.0 🆕)

### read 工具格式路由

```typescript
// read.ts

// 格式检测表
const OFFICE_FORMATS: Record<string, string> = {
  xlsx: "xlsx", xlsm: "xlsx",
  docx: "docx", pptx: "pptx", pdf: "pdf",
};

function detectFormat(filePath: string): string {
  const ext = extname(filePath).toLowerCase().slice(1);
  if (ext === "md" || ext === "markdown") return "markdown";
  return OFFICE_FORMATS[ext] || "text";
}

// execute 中的调度逻辑
const format = detectFormat(resolvedPath);
if (OFFICE_FORMATS[format]) {
  // Python 提取 → 统一返回 text
  const { text, error } = await extractWithPython(resolvedPath, format);
  // ... 构建 ReadResult
}
// 否则走纯文本读取逻辑
```

### Python 提取器模板

每个 Office 格式对应一个独立的 Python 提取脚本，通过 `sys.argv[1]` 传入文件路径：

```
提取脚本 = pipBootstrap([所需包]) + 具体提取逻辑
```

- Excel: `openpyxl.load_workbook()` → 遍历 sheets → `\\t` 分隔列
- Word: `python-docx` → 提取 paragraphs + tables
- PPT: `python-pptx` → 遍历 slides → 提取 text_frame
- PDF: `pdfplumber` 优先 → `PyPDF2` 备用

⚠️ **注意**：文件路径中有 Windows 反斜杠，Python 脚本用 raw string `r"..."` 包裹。

### write 工具 Office 写入模式

```typescript
// write.ts - Office 写入 dispatch

const format = detectWriteFormat(resolvedPath);
if (format === "xlsx" || format === "docx" || format === "pptx") {
  if (append) {
    return { error: "Append not supported for Office formats" };
  }
  // 内容 Base64 编码 → 嵌入 Python 脚本 → 执行
  const b64 = Buffer.from(content, "utf-8").toString("base64");
  const script = getXlsxScript(resolvedPath, b64);  // 或 getDocxScript / getPptxScript
  const result = await runPythonConversion(script, workspace);
}
```

⚠️ **关键**：内容通过 Base64 编码传递给 Python 脚本，避免 shell 特殊字符（引号、换行）导致的转义问题。

## exec 工具 Windows 编码处理模式 (v0.3.0 🆕)

### 问题：chcp 65001 对 pipe 输出无效

⚠️ **核心认知**：`chcp 65001` 只改变控制台(Console)的代码页，对 pipe (stdio: "pipe") 输出完全无效。必须在 Node.js 接收端用 `TextDecoder('gbk')` 解码。

### 正确实现

```typescript
// exec.ts - 编码处理

function decodeBuffer(buf: Buffer): string {
  if (buf.length === 0) return "";
  // 先用 UTF-8 尝试解码
  const utf8 = buf.toString("utf-8");
  // 如果 UTF-8 解码没有产生替换字符 (U+FFFD)，说明是合法 UTF-8
  if (!utf8.includes("\ufffd")) return utf8;
  // UTF-8 解码有异常 → 尝试 GBK
  try {
    return new TextDecoder("gbk", { fatal: false }).decode(buf);
  } catch {
    return utf8;
  }
}

// 在 child_process.exec 中使用
const child = exec(command, {
  shell: getShell(),
  timeout: timeout * 1000,
  maxBuffer: 1024 * 1024,
  encoding: "buffer" as BufferEncoding,  // 🔑 获取原始 Buffer
}, (error, stdout, stderr) => {
  const decodedStdout = decodeBuffer(stdout);
  const decodedStderr = decodeBuffer(stderr);
  // ... use decodedStdout/decodedStderr
});
```

### 验证方法

```
# 修复前：
echo 中文测试  →  ���Ĳ���   (GBK 被 UTF-8 解码，乱码)

# 修复后：
echo 中文测试  →  中文测试   (GBK 正确解码)
```

### exec 危险命令黑名单（平台感知版）

```typescript
function getBlacklist(): RegExp[] {
  const common: RegExp[] = [
    /^rm\s+-rf\s+\/\s*$/,
    /^rm\s+-rf\s+\/\*$/,
    /^chmod\s+-r\s+777\s+\//,
    /^mkfs\.[a-z0-9]+\s+/,
    /^dd\s+if=/,
  ];

  if (platform() === "win32") {
    return [
      ...common,
      /^del\s+\/f(\s+\/s)?/,
      /^format\s+/,
      /^diskpart(\s+|$)/,
      /^reg\s+delete\s+/,
      /^taskkill\s+\/f\s+/,
      /^shutdown\s+/,
      /^sudo\s+/,  // ⚠️ Windows 11 原生支持 sudo
    ];
  }
  
  return [
    ...common,
    /^sudo\s+/,
    /^shutdown/,
    /^reboot/,
    /^init\s+[06]/,
  ];
}
```

⚠️ **关键**：
- `encoding: "buffer"` 是解决编码问题的关键——不使用 buffer 则 GBK 解码无从谈起
- `TextDecoder('gbk')` 作为 UTF-8 失败后的 fallback，不改变正常 UTF-8 输出
- Windows 11 黑名单需包含 `sudo`（系统已原生支持）

## download 工具模式 (v0.3.0 🆕)

### 流式下载 + SSRF 防护

```typescript
// download.ts - 文件下载（支持 PDF、图片等任意类型）

import { createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";
import { Readable } from "node:stream";

// SSRF 防护（复用 web_fetch 黑名单）
const BLOCKED_PREFIXES = ["0.", "10.", "100.", "127.", "169.254.", "172.16.", ..., "192.168."];

function isBlockedHost(hostname: string): boolean { /* ... */ }

// 文件名提取（Content-Disposition 优先 → URL 路径 → 时间戳回退）
function extractFilename(url: string, contentDisposition: string | null): string {
  if (contentDisposition) {
    const fnMatch = contentDisposition.match(/filename\*?=(?:UTF-8''|"?)([^";\s]+)/i);
    if (fnMatch) return decodeURIComponent(fnMatch[1]);
  }
  return basename(new URL(url).pathname) || `download_${Date.now()}`;
}

// 重名自动追加 _(1) _(2)
function ensureUnique(filepath: string): string { /* ... */ }

export async function execute(args: Record<string, unknown>): Promise<DownloadResult> {
  const url = args["url"] as string;
  const response = await fetch(url, { headers: { "User-Agent": "..." }, redirect: "follow" });
  
  // 流式写入，避免内存溢出
  const nodeStream = Readable.fromWeb(response.body as any);
  await pipeline(nodeStream, createWriteStream(savePath));
  
  return { success: true, path: savePath, size: stats.size, contentType };
}
```

⚠️ **关键**：
- `Readable.fromWeb()` 将 Web Streams API 的 ReadableStream 转为 Node.js Readable
- `pipeline()` 自动管理背压和清理，比手动 pipe 更可靠
- 超时 60s（比 web_fetch 的 15s 更长，适合大文件）
- 文件名处理：`Content-Disposition` 头部优先，支持 `filename*` (RFC 5987) 和普通 `filename`

## web_search 三引擎并行模式 (v0.3.0 🆕)

### Promise.allSettled 并行 + 结果合并去重

```typescript
// web_search.ts - Bing + Google + 百度 同时发起，合并去重

async function searchBing(query: string, count: number): Promise<SearchResultItem[]> { /* cn.bing.com */ }
async function searchGoogle(query: string, count: number): Promise<SearchResultItem[]> { /* 需代理 */ }
async function searchBaidu(query: string, count: number): Promise<SearchResultItem[]> { /* 国内直连 */ }

function normalizeUrl(u: string): string {
  // 去掉 www. 前缀和尾部斜杠，用于去重比较
  return new URL(u).hostname.replace(/^www\./, "") + new URL(u).pathname.replace(/\/$/, "");
}

export async function execute(args: Record<string, unknown>): Promise<SearchResult> {
  // 三引擎同时发起（并行，不是 fallback）
  const [bingRes, googleRes, baiduRes] = await Promise.allSettled([
    searchBing(query, count),
    searchGoogle(query, count),
    searchBaidu(query, count),
  ]);

  // 收集成功的结果
  const allResults: SearchResultItem[] = [];
  if (bingRes.status === "fulfilled") allResults.push(...bingRes.value);
  if (googleRes.status === "fulfilled") allResults.push(...googleRes.value);
  if (baiduRes.status === "fulfilled") allResults.push(...baiduRes.value);

  // URL 去重（保留首次出现）
  const seen = new Set<string>();
  const deduped = allResults.filter(r => {
    const key = normalizeUrl(r.url);
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });

  return { success: true, results: deduped, query, engines: usedEngines };
}
```

⚠️ **关键**：
- `Promise.allSettled` 不因单个引擎失败而影响其他引擎
- 合并结果后按 URL 主机名+路径去重，避免同一页面在多个引擎出现
- 返回 `engines` 字段，标记哪些引擎贡献了结果（方便调试）
- Google 需要代理才能通，Bing 用 `cn.bing.com` 国内直连，百度无需代理
