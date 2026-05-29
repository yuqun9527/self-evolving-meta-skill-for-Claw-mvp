# Claw MVP — 系统设计文档

> **版本**: v2.0  
> **作者**: 架构设计师 (Designer)  
> **状态**: 设计完成，已实现

---

## 1. 项目概述

### 1.1 一句话定义

**claw** 是一个单进程 CLI + Web 双模式 AI Agent 工具。TypeScript (ESM)，Node.js 22.6+ 用 `--experimental-strip-types` 直接运行，零编译。用户通过终端或浏览器输入自然语言指令，claw 驱动 LLM 调用内置工具（读文件、写文件、执行命令、搜索网络等）完成半自动化任务。

### 1.2 核心价值主张

| 维度 | 描述 |
|------|------|
| **轻量零编译** | `node --experimental-strip-types src/index.ts` 直接启动，单依赖 `openai`，无 tsc/tsx |
| **安全沙箱** | 工作区隔离 + exec 危险命令黑名单 + SSRF 内网拦截 + 反 LLM 幻觉规则 |
| **办公原生** | .xlsx/.docx/.pptx/.pdf 自动解析与生成，Python 脚本执行 + 自动 pip install |
| **双入口模式** | CLI REPL + Web 聊天界面，共享同一 Agent 核心 |
| **多引擎搜索** | Bing + Google + 百度三引擎并行，结果合并去重 |
| **多 Provider** | 原生支持 DeepSeek、OpenAI 等兼容接口 |

### 1.3 目标用户画像

- **个人开发者**：日常要用 AI 辅助读取/编辑代码文件、执行调试命令
- **CLI 重度用户**：习惯终端操作，需要快速 AI 辅助能力
- **Web 偏好者**：喜欢浏览器聊天界面，需要流式输出和可视化工具调用
- **私有化偏好者**：数据不想经过 Web UI，希望所有交互在本地终端完成

### 1.4 设计原则

| 原则 | 说明 |
|------|------|
| **最小可用** | 只做 MVP 必需功能，不加额外抽象 |
| **零编译** | `--experimental-strip-types` 直接运行 .ts 文件 |
| **零外部运行时依赖** | 不依赖 Docker、Redis、数据库等 |
| **透明可审计** | 所有交互显示在终端/Web，工具调用过程可见 |
| **可移植** | 支持 Windows / macOS / Linux 三大平台 |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌───────────────────────────────────────────────────────────────────────┐
│                         双入口 (Dual Entry)                           │
│  ┌──────────────────────────┐    ┌──────────────────────────────┐    │
│  │ src/index.ts (CLI)      │    │ src/server.ts (Web)          │    │
│  │ readline REPL 循环       │    │ node:http + SSE 流式聊天     │    │
│  │ 斜杠命令路由 + stdout    │    │ REST API + 多会话管理       │    │
│  └───────────┬──────────────┘    └────────────┬─────────────────┘    │
│              │                                │                       │
│              └──────────┬─────────────────────┘                       │
│                         ▼                                             │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ src/agent.ts  (Agent 主循环)                                  │    │
│  │                                                               │    │
│  │  1. buildMessages() — 消息截断 + tool_calls/tool 配对保护     │    │
│  │  2. LLMClient.chat() — 流式生成 + tools 定义传入              │    │
│  │  3. consumeStream() — reasoning/text/tool_call delta 分发     │    │
│  │  4. 文本响应 → 反空话检测 → 正常则结束                        │    │
│  │  5. tool_calls → 搜索轮次检查(≤4) → 执行工具 → 结果回传      │    │
│  │  6. 重复 2-5，最多 maxIterations=30 轮                        │    │
│  └──────────┬──────────────────────────────────────────────┘    │
│             │                                                    │
│     ┌───────┼────────── LLM 调用 ───────────┐                    │
│     │       ▼                                │                    │
│     │  ┌─────────────────────┐              │                    │
│     │  │ src/llm.ts          │              │                    │
│     │  │ OpenAI SDK 封装     │              │                    │
│     │  │ 流式 API + tools    │              │                    │
│     │  │ reasoning_content   │              │                    │
│     │  │ Provider 路由       │              │                    │
│     │  └──────┬──────────────┘              │                    │
│     │         │ HTTPS                       │                    │
│     │         ▼                             │                    │
│     │  ┌──────────────┐                    │                    │
│     │  │ LLM API      │                    │                    │
│     │  └──────────────┘                    │                    │
│     │                                      │                    │
│     └────────── 工具调用 ───────────────────┘                    │
│                │                                                 │
│                ▼                                                 │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ src/tools/index.ts  (工具注册中心 — 7 工具)                │   │
│  │                                                           │   │
│  │  tools:                                                   │   │
│  │   ├── read        → 文件读取 (含 Office/PDF 提取)        │   │
│  │   ├── write       → 文件写入 (含 Office 自动转换)        │   │
│  │   ├── edit        → 精确文本替换                          │   │
│  │   ├── exec        → 命令执行 (跨平台 + 编码修复 + 安全)   │   │
│  │   ├── web_search  → Bing + Google + 百度 三引擎并行      │   │
│  │   ├── web_fetch   → URL 内容抓取 (SSRF 防护)             │   │
│  │   └── python      → Python 脚本执行 (auto-pip + 安全检查) │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ src/config.ts   → 配置加载 (默认 → config.json → 环境变量) │   │
│  │ src/session.ts  → 会话持久化 (save/load/list/delete)       │   │
│  │ system-prompt.md → System Prompt 独立配置文件              │   │
│  └───────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘

Web 前端:
┌──────────────────────────────────────────────────────────────────┐
│  public/index.html  (SPA 聊天界面)                               │
│  ├─ 暗色主题，类似 OpenClaw 风格                                  │
│  ├─ EventSource SSE 接收流式输出                                  │
│  ├─ 绿色标签显示工具调用 (名称 + 参数)                            │
│  ├─ 会话管理 (URL ?session=xxx + 新建/清空按钮)                  │
│  └─ 模型切换下拉框                                                │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 模块划分和职责

| 模块 | 文件 | 职责 |
|------|------|------|
| 说明文档 | `README.md` | 项目说明文档（内容以实际代码为准，结构参考 README.template.md） |
| CLI 入口 | `src/index.ts` | REPL 循环、10 种斜杠命令、启动 Banner |
| Web 服务 | `src/server.ts` | HTTP + SSE 流式聊天、REST API、多会话管理 |
| 前端界面 | `public/index.html` | SPA 聊天界面、流式显示、工具调用可视化 |
| Agent 运行器 | `src/agent.ts` | Agent Loop 编排、consumeStream、搜索限制、反空话 |
| LLM 调用 | `src/llm.ts` | OpenAI SDK 封装、Streaming、reasoning_content、Provider 路由 |
| 工具注册中心 | `src/tools/index.ts` | 工具定义注册、模块化 ToolRegistry |
| 文件读取 | `src/tools/read.ts` | 文本 + Office/PDF 格式自动提取 |
| 文件写入 | `src/tools/write.ts` | 文本 + Office 格式自动转换 |
| 文本编辑 | `src/tools/edit.ts` | 精确文本替换（唯一性校验） |
| 命令执行 | `src/tools/exec.ts` | Shell 执行 + 安全拦截 + 编码修复 |
| 网络搜索 | `src/tools/web_search.ts` | 三引擎并行搜索（Bing + Google + 百度，结果合并去重） |
| 网页抓取 | `src/tools/web_fetch.ts` | HTTP/HTTPS 抓取 + SSRF 防护 |
| 文件下载 | `src/tools/download.ts` | 流式下载到工作区（PDF、图片等任意类型，SSRF 防护） |
| Python 执行 | `src/tools/python.ts` | Python 脚本执行 + auto-pip + 安全检查 |
| 配置管理 | `src/config.ts` | 配置加载与合并 + System Prompt 占位符替换 |
| 会话管理 | `src/session.ts` | 会话序列化与持久化 + 索引兼容 |
| System Prompt | `system-prompt.md` | 独立配置文件（中文 Markdown，安全策略 + 操作规则） |

### 2.3 数据流

```
用户输入 (CLI stdin 或 Web POST /api/chat)
  │
  ▼
[入口层] ─── CLI 斜杠命令? ───→ /save /load /clear /model ...
  │
  ▼ 自然语言消息
[Agent Loop — agent.ts]
  │
  ├─1. buildMessages() → system prompt + 截断历史 (maxHistoryTurns=50)
  │
  ├─2. LLMClient.chat() → streaming + tools 定义
  │     │
  │     ├─ text_delta → 流式输出 (stdout 或 SSE onStream)
  │     │   └─ done → consumeStream 返回 text
  │     │
  │     └─ tool_call_delta → 累积
  │         └─ done(finish_reason=tool_calls) → consumeStream 返回 tool_calls
  │
  ├─3. 文本响应 → 反空话检测 (looksLikeChatter)
  │     ├─ 空话且重试 <3 → 注入 SYSTEM 消息 → 回到步骤 2
  │     └─ 正常 → 存储 assistant 消息 → 结束
  │
  ├─4. tool_calls → 搜索轮次检查 (searchRoundCount ≤ 4)
  │     ├─ 超限 → 注入搜索错误 → 继续
  │     └─ 正常 → 逐个执行工具 → 结果回传消息
  │
  └─5. iteration++ → 回到步骤 2 (最多 30 轮)
```

### 2.4 模块间调用关系

```
index.ts ──→ config.ts (启动加载)
  ├──→ session.ts (save/load/list/delete 斜杠命令)
  └──→ agent.ts (处理用户自然语言消息)
          ├──→ llm.ts (Agent 循环中调用 LLM)
          └──→ tools/ (Agent 循环中调用工具)

server.ts ──→ config.ts (启动加载)
  ├──→ session.ts (持久化会话操作)
  ├──→ agent.ts (处理用户消息 + AgentCallbacks 注入)
  └──→ public/index.html (静态文件服务)
```

---

## 3. 模块详细设计

### 3.1 CLI 入口模块 (`src/index.ts`)

#### 3.1.1 REPL 生命周期

```
┌─────────────────────────────────────┐
│  启动                                 │
│  ├─ loadConfig()                     │
│  ├─ 创建 workspace 目录              │
│  ├─ 打印启动 Banner                  │
│  └─ 初始化 Agent                     │
│                                       │
│  ↓                                    │
│  REPL 主循环                          │
│  ├─ readline.question("gr> ")         │
│  ├─ 空输入 → 跳过                    │
│  ├─ 斜杠命令? → 路由分发             │
│  │   ├─ /file   → 读取文件注入上下文  │
│  │   ├─ /model  → 切换模型           │
│  │   ├─ /clear  → 清屏 + 清上下文    │
│  │   ├─ /history→ 打印消息历史       │
│  │   ├─ /save   → session.save()     │
│  │   ├─ /load   → session.load()     │
│  │   ├─ /list   → session.list()     │
│  │   ├─ /delete → session.delete()   │
│  │   ├─ /exit   → 退出               │
│  │   └─ /help   → 打印帮助           │
│  └─ 自然语言 → agent.run(text)       │
│                                       │
│  ↓ (用户 /exit 或 Ctrl+C×2)          │
│  process.exit(0)                      │
└─────────────────────────────────────┘
```

#### 3.1.2 斜杠命令

| 命令 | 参数 | 说明 |
|------|------|------|
| `/file <path>` | 文件路径 | 将文件内容注入到当前上下文 |
| `/model <name>` | 模型名称 | 切换当前模型 |
| `/clear` | — | 清屏 + 清空消息历史 |
| `/history` | — | 打印消息历史摘要 |
| `/save [name]` | 可选会话名 | 保存当前会话 |
| `/load <name>` | 会话名 | 加载历史会话 |
| `/list` | — | 列出所有已保存会话 |
| `/delete <name>` | 会话名 | 删除指定会话 |
| `/exit` | — | 退出程序 |
| `/help` | — | 打印帮助信息 |

#### 3.1.3 交互设计

- 使用 Node.js 原生 `readline` 模块
- Tab 自动补全斜杠命令
- Ctrl+C 一次中断当前生成，两次退出程序
- 流式输出通过 `process.stdout.write` 实时显示

### 3.2 Web 服务模块 (`src/server.ts`)

#### 3.2.1 整体设计

- 基于原生 `node:http` 创建 HTTP 服务器，零框架依赖
- SSE (Server-Sent Events) 实现流式聊天推送
- 内存 Map 管理多会话（每个 sessionId 对应独立 Agent 实例）
- 静态文件服务支持前端 SPA

#### 3.2.2 端口解析

优先级（从高到低）：
```
CLI 参数 --port/-p/纯数字 > 环境变量 CLAW_PORT > 默认 3456
```

示例：
```bash
npm run web -- --port 8080
npm run web -- -p 8080
npm run web -- 8080
CLAW_PORT=8080 npm run web
```

#### 3.2.3 REST API

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/chat` | 发送消息，SSE 流式返回 |
| `GET` | `/api/sessions` | 列出所有会话（活跃 + 持久化） |
| `POST` | `/api/sessions` | 创建新会话 |
| `POST` | `/api/sessions/load` | 加载持久化会话到内存 |
| `POST` | `/api/sessions/:id/save` | 保存会话到磁盘 |
| `DELETE` | `/api/sessions/:id` | 删除会话 |
| `GET` | `/api/sessions/persisted` | 列出磁盘持久化会话 |
| `GET` | `/api/config` | 获取当前配置 |
| `GET` | `/api/models` | 获取可用模型列表 |
| `POST` | `/api/model` | 切换模型 |

#### 3.2.4 SSE 事件类型

| 事件 | 数据 | 说明 |
|------|------|------|
| `session` | `{ sessionId, model, workspace }` | 会话信息（首次推送） |
| `text` | `{ content }` | 流式文本块 |
| `tool_call` | `{ name, args }` | 工具调用通知 |
| `tool_result` | `{ name, result }` | 工具执行结果 |
| `error` | `{ message }` | 错误消息 |
| `done` | `{ sessionId }` | 请求完成 |

#### 3.2.5 多会话管理

```typescript
interface SessionAgent {
  agent: Agent;
  config: ClawConfig;
  createdAt: number;
}

const sessions = new Map<string, SessionAgent>();
```

- 按 `sessionId` 隔离 Agent 实例
- 默认会话 ID: `"default"`
- 支持动态创建、切换、删除
- Web 界面通过 URL 参数 `?session=xxx` 切换

#### 3.2.6 静态文件服务

- `public/index.html` — 聊天界面 SPA
- MIME 类型映射：`.html` `.css` `.js` `.json` `.png` `.svg` `.ico`
- 根路径 `/` → `/index.html`
- CORS 全开（本地开发场景）

### 3.3 Agent 运行器模块 (`src/agent.ts`)

#### 3.3.1 核心类型

```typescript
// LLM 流式块
type LLMChunk =
  | { type: "reasoning_delta"; delta: string }
  | { type: "text_delta"; delta: string }
  | { type: "tool_call_delta"; index: number; delta: {...} }
  | { type: "done"; finish_reason: string };

// Agent 回调接口（Web 模式用回调代替 stdout）
interface AgentCallbacks {
  onStream?: (text: string) => void;
  onToolCall?: (name: string, args: Record<string, unknown>) => void;
  onToolResult?: (name: string, result: unknown) => void;
  onError?: (msg: string) => void;
}
```

#### 3.3.2 Agent Loop 伪代码

```typescript
class Agent {
  messages: Message[] = [];
  config: ClawConfig;
  tools: ToolRegistry;
  llmClient: LLMClient;
  searchRoundCount = 0;       // 搜索轮次计数器
  chatterRetryCount = 0;      // 反空话重试计数
  abortRequested = false;     // 中断标志
  generating = false;         // 正在生成标志
  callbacks: AgentCallbacks;  // 回调接口

  async run(userInput: string): Promise<void> {
    this.messages.push({ role: "user", content: userInput });
    let iteration = 0;

    while (iteration < this.config.maxIterations && !this.abortRequested) {
      iteration++;

      // 1. 调用 LLM (streaming + tools)
      const generator = this.llmClient.chat(
        this.buildMessages(),
        undefined, // signal
        this.tools.getDefinitions()
      );

      // 2. 消费流
      const result = await this.consumeStream(generator);

      // 3. 文本响应处理
      if (result.type === "text") {
        // 反空话检测
        if (isEmptyOrChatter(result.content)) {
          if (chatterRetryCount++ > 3) break; // 放弃
          // 注入 SYSTEM 消息要求重新回答
          continue;
        }
        // 正常 → 存储→ 结束
        this.messages.push({ role: "assistant", content: result.content, reasoning_content: result.reasoning_content });
        break;
      }

      // 4. tool_calls 处理
      if (result.type === "tool_calls") {
        // 搜索轮次检查
        this.searchRoundCount += countSearchTools(result.toolCalls);
        if (this.searchRoundCount > 4) {
          // 注入搜索错误 → 继续循环
          continue;
        }

        // 存储 assistant(tool_calls)
        this.messages.push({ role: "assistant", content: null, tool_calls: result.toolCalls, reasoning_content: result.reasoning_content });

        // 执行每个工具 → 回传结果
        for (const tc of result.toolCalls) {
          const result = await this.tools.execute(tc.function.name, JSON.parse(tc.function.arguments));
          this.messages.push({ role: "tool", tool_call_id: tc.id, content: JSON.stringify(result) });
        }
      }
    }
  }
}
```

#### 3.3.3 consumeStream() — 流消费

```typescript
private async consumeStream(generator): Promise<LLMFinalResult> {
  let textContent = "";
  let reasoningContent = "";
  const toolCallAcc = new Map<number, ToolCallDelta>();

  for await (const chunk of generator) {
    if (this.abortRequested) break;

    switch (chunk.type) {
      case "reasoning_delta":
        reasoningContent += chunk.delta;
        break;
      case "text_delta":
        textContent += chunk.delta;
        this.callbacks.onStream?.(chunk.delta);
        break;
      case "tool_call_delta":
        // 按 index 累积 tool call delta
        break;
      case "done":
        if (chunk.finish_reason === "tool_calls" && toolCallAcc.size > 0) {
          return { type: "tool_calls", toolCalls: [...], reasoning_content: reasoningContent };
        }
        return { type: "text", content: textContent, reasoning_content: reasoningContent };
    }
    // ❌ 绝不能在此处加 return（BUG-006：会导致首 chunk 后退出）
  }

  return { type: "text", content: textContent, reasoning_content: reasoningContent };
}
```

#### 3.3.4 buildMessages() — 消息构建与截断

- 最前插入 system prompt（含 `{workspace}` 实时替换）
- 截断到最近 `maxHistoryTurns` 条消息（默认 50）
- **截断保护**：如果截断后首条是 tool 消息，向前回溯到最近的 assistant(tool_calls)
- 保证发送给 LLM 的消息中 tool_calls/tool 配对完整

#### 3.3.5 搜索轮次限制

- `web_search` 和 `web_fetch` 合并计数
- 每轮最多 4 次搜索相关工具调用
- 超限后：对搜索工具注入错误信息，非搜索工具正常执行
- 确保 Assistant 消息先于 Tool 消息写入

#### 3.3.6 反空话检测

- 4 种正则模式检测"只说不做"型响应
- 空文本也触发重试
- 最多 3 次重试 → 放弃（防止死循环）
- 重试时注入 SYSTEM 消息要求重新回答

#### 3.3.7 中断处理

- `abortRequested` 标志位中断 LLM 流式生成
- `generating` 标志位防止 Web 模式并发冲突
- CLI：Ctrl+C 设置 abortRequested
- Web：新消息到达时自动中断上一轮生成

### 3.4 LLM 模块 (`src/llm.ts`)

#### 3.4.1 核心类型

```typescript
interface Message {
  role: "system" | "user" | "assistant" | "tool";
  content: string | null;
  tool_calls?: ToolCall[];
  tool_call_id?: string;
  name?: string;
  reasoning_content?: string;  // DeepSeek V4 推理内容
}

interface ToolCall {
  id: string;
  type: "function";
  function: {
    name: string;
    arguments: string;  // JSON 字符串
  };
}
```

#### 3.4.2 OpenAI SDK 封装

```typescript
class LLMClient {
  private client: OpenAI;
  private model: string;

  constructor(config: ProviderConfig, model: string) {
    this.client = new OpenAI({
      baseURL: config.baseURL,   // 如 https://api.deepseek.com/v1
      apiKey: config.apiKey,
      timeout: (config.timeout || 60) * 1000,
    });
  }

  async *chat(
    messages: Message[],
    signal?: AbortSignal,
    tools?: OpenAI.Chat.Completions.ChatCompletionTool[]
  ): AsyncGenerator<LLMChunk>
}
```

#### 3.4.3 多 Provider 支持

```
模型名称格式: [provider/]model-name

示例:
  "deepseek/deepseek-v4-flash"  → provider=deepseek, model=deepseek-v4-flash
  "deepseek/deepseek-chat"      → provider=deepseek, model=deepseek-chat
  "deepseek/deepseek-reasoner"  → provider=deepseek, model=deepseek-reasoner
```

#### 3.4.4 DeepSeek V4 reasoning_content 全链路

```
LLM API 响应
  │ delta.reasoning_content
  ▼
chat() 捕获 → yield { type: "reasoning_delta", delta: rc }
  │
  ▼
consumeStream() 累积 → LLMFinalResult.reasoning_content
  │
  ▼
run() 存储 assistant 消息 → Message.reasoning_content
  │
  ▼
buildMessages() → toOpenAIMessages()
  │ 附加到 OpenAI 消息 (base as Record<string,unknown>).reasoning_content
  ▼
传给 LLM API（DeepSeek V4 必须传回）
```

#### 3.4.5 toOpenAIMessages() — 消息映射

- `role: "system"|"user"` → 直接映射 content
- `role: "assistant"` → 映射 content + tool_calls + reasoning_content
- `role: "tool"` → 映射 content + tool_call_id
- 所有 assistant 消息必须携带 `reasoning_content`（存了就必须传回）

### 3.5 工具模块 (`src/tools/`)

#### 3.5.1 工具注册中心

```typescript
class ToolRegistry {
  private tools: Map<string, RegisteredTool>;
  private config: ClawConfig;

  constructor(config: ClawConfig) {
    this.config = config;
    // 模块化注册
    this.register(readTool);
    this.register(writeTool);
    this.register(editTool);
    this.register(execTool);
    this.register(webSearchTool);
    this.register(webFetchTool);
    this.register(pythonTool);
  }

  getDefinitions(): Array<{ type: "function"; function: Record<string, unknown> }>
  async execute(name: string, args: Record<string, unknown>): Promise<Record<string, unknown>>
}
```

#### 3.5.2 工具：`read` — 文件读取

| 项目 | 描述 |
|------|------|
| **工具名** | `read` |
| **参数** | `path` (必填), `offset` (可选), `limit` (可选) |
| **支持格式** | 纯文本 (.txt .md .csv .json .html .xml .yaml .py .ts .js 等) |
| **Office/PDF** | .xlsx (openpyxl) .docx (python-docx) .pptx (python-pptx) .pdf (pdfplumber/PyPDF2) |
| **路径** | 相对路径→workspace，自动去除 "workspace/" 前缀 |
| **安全** | UNC 拦截、跨盘符拒绝、路径遍历拒绝 |
| **大小限制** | 纯文本 ≤1MB，Office/PDF ≤10MB |

#### 3.5.3 工具：`write` — 文件写入

| 项目 | 描述 |
|------|------|
| **工具名** | `write` |
| **参数** | `path` (必填), `content` (必填), `append` (可选) |
| **纯文本** | UTF-8 直接写入，支持追加模式 |
| **Office 转换** | .xlsx (CSV/TSV→Excel) .docx (Markdown→Word) .pptx (# 标题→幻灯片) |
| **安全** | UNC 拦截、跨盘符拒绝、路径遍历拒绝 |
| **注意** | Office 格式不支持追加模式 |

#### 3.5.4 工具：`edit` — 精确文本替换

| 项目 | 描述 |
|------|------|
| **工具名** | `edit` |
| **参数** | `path`, `oldText`, `newText` (均必填) |
| **唯一性校验** | 匹配到 0 处→报错，匹配 >1 处→报错（要求更具体文本） |
| **大小限制** | 编辑后文件 ≤5MB |
| **安全** | 同 read/write 的工作区边界检查 |

#### 3.5.5 工具：`exec` — Shell 命令执行

| 项目 | 描述 |
|------|------|
| **工具名** | `exec` |
| **参数** | `command` (必填), `timeout` (可选, 默认30s, 最大300s) |
| **平台** | Windows: PowerShell, 其他: bash |
| **编码处理** | `encoding: "buffer"` + UTF-8 尝试 → GBK 回退（TextDecoder） |
| **安全检查** | 多层：黑名单(rm -rf / / format / shutdown / sudo 等) + 下载执行链 + PowerShell编码命令 + 环境变量泄露 + 文件操作沙箱 |
| **文件操作沙箱** | 检测 Copy-Item/Move-Item/copy/move/xcopy/robocopy 目标 + 重定向 > >> 目标 |

#### 3.5.6 工具：`web_search` — 网络搜索

| 项目 | 描述 |
|------|------|
| **工具名** | `web_search` |
| **参数** | `query` (必填), `count` (可选, 默认5, 最大10) |
| **引擎** | Bing (cn.bing.com) + Google + 百度，三引擎并行 (`Promise.allSettled`)，结果合并去重 |
| **限制** | 每任务最多 4 轮（与 web_fetch 合并计数） |

#### 3.5.7 工具：`web_fetch` — 网页抓取

| 项目 | 描述 |
|------|------|
| **工具名** | `web_fetch` |
| **参数** | `url` (必填), `maxChars` (可选, 默认5000) |
| **SSRF 防护** | IPv6 mapped IPv4 检测、十进制/十六进制 IP 检测、169.254.0.0/16 拦截、file:// 拒绝 |
| **限制** | 仅 HTTP/HTTPS |

#### 3.5.8 工具：`python` — Python 脚本执行

| 项目 | 描述 |
|------|------|
| **工具名** | `python` |
| **参数** | `script` (必填), `timeout` (可选, 默认60s, 最大300s), `packages` (可选额外 pip 包) |
| **auto-pip** | 自动安装核心包（openpyxl/python-docx/python-pptx/pdfplumber/PyPDF2） |
| **包名映射** | `python-docx`→`docx`，`python-pptx`→`pptx`（不能依赖 replace("-","_")） |
| **安全检查** | 拦截 os/subprocess/socket/ctypes/eval/exec/requests/urllib/http.client/shutil + `__import__('os')` |
| **编码** | PYTHONIOENCODING=utf-8 + PYTHONUTF8=1 |
| **临时文件** | workspace 内 `.claw-python-{uuid}.py`，执行后自动清理 |

### 3.6 配置模块 (`src/config.ts`)

#### 3.6.1 配置接口

```typescript
interface ProviderConfig {
  baseURL: string;
  apiKey: string;
  timeout?: number;
}

interface ClawConfig {
  model: string;                          // 如 "deepseek/deepseek-v4-flash"
  providers: Record<string, ProviderConfig>;
  workspace: string;                      // 工作区目录（绝对路径）
  maxHistoryTurns: number;                // 最大消息条数，默认 50
  maxIterations: number;                  // 最大工具调用轮数，默认 30
  systemPrompt: string;                   // system prompt 内容
}
```

#### 3.6.2 配置优先级

```
环境变量 (CLAW_MODEL / CLAW_WORKSPACE / DEEPSEEK_API_KEY)
          │ 覆盖
          ▼
~/.claw/config.json
          │ 覆盖
          ▼
DEFAULT_CONFIG (硬编码默认值)
```

#### 3.6.3 System Prompt 加载优先级

```
config.json systemPrompt 字段 > system-prompt.md > 内置回退
```

- `system-prompt.md` 支持占位符替换：`{workspace}` / `{platform}` / `{shell}`
- 内置回退为最精简提示词

### 3.7 会话持久化模块 (`src/session.ts`)

#### 3.7.1 存储结构

```
~/.claw/sessions/
  ├── index.json          # 索引: { nameToId: { name: uuid }, ids: string[] }
  ├── <uuid1>.json        # 会话数据: SessionData
  └── <uuid2>.json
```

#### 3.7.2 数据接口

```typescript
interface SessionData {
  id: string;              // UUID
  name: string;            // 用户命名或自动生成
  createdAt: string;       // ISO 时间戳
  updatedAt: string;
  model: string;
  messages: Message[];     // 不含 system 消息
}

interface SessionIndex {
  nameToId: Record<string, string>;  // name → uuid 映射
  ids: string[];                      // 所有 uuid 列表
}
```

#### 3.7.3 向后兼容

- `loadIndex()` 检测 `parsed.nameToId` 是否存在
- 旧格式（扁平 `{name: id}`）自动转换为新格式

---

## 4. 安全架构

### 4.1 纵深防御模型

```
┌──────────────────────────────────────────────┐
│ 第一层：System Prompt 语义约束（LLM 自身）     │
│ - workspace 是唯一可操作目录                   │
│ - 禁止 exec 文件传输                          │
│ - 禁止读取工作区外的路径                       │
│ - 外部路径请求 → 说明限制并建议替代方案        │
├──────────────────────────────────────────────┤
│ 第二层：工具层代码安全拦截（Tools）            │
│ - read/write/edit: workspace 边界检查          │
│ - exec: 黑名单 + 下载链 + 编码命令 + 文件沙箱   │
│ - web_fetch: SSRF 多层防御                     │
│ - python: 危险模块拦截 + 动态导入检测          │
├──────────────────────────────────────────────┤
│ 第三层：Agent 行为约束（Agent Loop）           │
│ - 搜索轮次上限可配置（默认 4，web_search + web_fetch） │
│ - 反空话检测 ≤3 次重试                        │
│ - maxIterations = 30 强制终止                  │
│ - abortRequested 中断机制                      │
└──────────────────────────────────────────────┘
```

### 4.2 安全拦截清单

| 类别 | 拦截项 |
|------|--------|
| 工作区沙箱 | UNC 路径、跨盘符、路径遍历(..)、绝对路径 |
| 危险命令 | rm -rf /、format、shutdown、reboot、diskpart、sudo |
| 编码命令 | -EncodedCommand、-Enc、-ec |
| 下载执行链 | iex iwr、irm、Invoke-Expression、curl \| bash |
| 环境变量泄露 | Get-ChildItem env:、ls env: |
| 管理命令 | schtasks、netsh、net user |
| 权限修改 | icacls、cacls、takeown、reg add |
| exec 文件沙箱 | Copy-Item/Move-Item -Destination、copy/move/xcopy 目标、重定向 > >> |
| SSRF | IPv6 mapped IPv4、十进制/十六进制 IP、169.254.0.0/16、file:// |
| Python 安全 | os/subprocess/socket/ctypes/eval/exec/shutil、`__import__('os')` |

---

## 5. 项目配置

### 5.1 package.json

```json
{
  "name": "claw-mvp",
  "version": "0.2.0",
  "description": "CLI AI Agent",
  "type": "module",
  "scripts": {
    "start": "node --experimental-strip-types src/index.ts",
    "web": "node --experimental-strip-types src/server.ts",
    "check": "node --experimental-strip-types --check src/index.ts"
  },
  "dependencies": {
    "openai": "^4.0.0"
  }
}
```

**关键点：**
- 唯一运行时依赖：`openai`
- 无 `typescript`、`tsx`、`ts-node` 等 devDependency
- 无 `build`/`dev` 脚本
- `--experimental-strip-types` 直接运行 .ts 文件

### 5.2 config.json 示例

```json
{
  "model": "deepseek/deepseek-v4-flash",
  "providers": {
    "deepseek": {
      "baseURL": "https://api.deepseek.com/v1",
      "apiKey": "sk-xxx"
    }
  },
  "workspace": "./workspace",
  "maxHistoryTurns": 50,
  "maxIterations": 30
}
```

---

## 6. 目录结构

```
claw-mvp/
├── package.json          # 项目配置 (type: module, ESM)
├── system-prompt.md      # System Prompt 独立配置文件（中文 Markdown）
├── README.md             # 使用说明文档
├── public/               # Web 静态资源
│   └── index.html        # 聊天界面 SPA（暗色主题）
├── src/                  # 源代码
│   ├── index.ts          # CLI 入口（REPL 循环 + 10 种斜杠命令）
│   ├── server.ts         # Web 服务入口（HTTP + SSE + REST API）
│   ├── agent.ts          # Agent 主循环（consumeStream + 反空话 + 搜索限制）
│   ├── llm.ts            # LLM 调用封装（OpenAI SDK + DeepSeek V4）
│   ├── config.ts         # 配置加载（三级优先级 + 占位符替换）
│   ├── session.ts        # 会话持久化（CRUD + 向后兼容）
│   └── tools/            # 工具实现（7 工具）
│       ├── index.ts      # 工具注册中心
│       ├── read.ts       # 文件读取（含 Office/PDF 提取）
│       ├── write.ts      # 文件写入（含 Office 转换）
│       ├── edit.ts       # 文本编辑
│       ├── exec.ts       # Shell 执行（含编码修复 + 安全拦截）
│       ├── web_search.ts # 网页搜索（三引擎并行）
│       ├── web_fetch.ts  # 网页抓取（SSRF 防护）
│       ├── download.ts  # 文件下载（流式写入）
│       ├── web_fetch.ts  # 网页抓取（SSRF 防护）
│       └── python.ts     # Python 执行（auto-pip + 安全检查）
└── workspace/            # 默认工作区（gitignore）
```

---

## 7. 运行方式

### CLI 模式
```bash
npm start
# 等价于: node --experimental-strip-types src/index.ts
```

### Web 模式
```bash
npm run web
# 等价于: node --experimental-strip-types src/server.ts
# 浏览器打开 http://localhost:3456

# 自定义端口
npm run web -- --port 8080
```

### 类型检查
```bash
npm run check
# 等价于: node --experimental-strip-types --check src/index.ts
# 注意：--check 通过 ≠ 运行时无问题，必须实际启动验证
```

---

## 8. 技术栈

| 技术 | 用途 |
|------|------|
| **Node.js 22.6+** | 运行时，`--experimental-strip-types` 直接运行 TS |
| **TypeScript (ESM)** | 开发语言，零编译 |
| **OpenAI SDK v4** | LLM API 调用（唯一运行时依赖） |
| **node:http** | Web 服务器（零框架） |
| **SSE** | 流式推送协议 |
| **DeepSeek V4** | 默认 LLM（支持 reasoning_content） |
| **Python 3** | Office/PDF 文件处理（node:child_process 调用） |

---

## 9. 设计决策记录

| 决策 | 选择 | 原因 |
|------|------|------|
| 运行时 | `--experimental-strip-types` | 零编译，开发效率高 |
| Web 框架 | 原生 `node:http` | 零外部依赖，极致轻量 |
| 流式协议 | SSE | 比 WebSocket 简单，单向推送够用 |
| LLM SDK | openai v4 | DeepSeek 兼容 OpenAI 协议 |
| Office 处理 | Python 子进程 | Node.js 生态 Office 库不成熟 |
| 配置存储 | ~/.claw/config.json | 用户级配置，简单持久化 |
| 会话存储 | 文件系统 JSON | 无需数据库，数据可审计 |
| System Prompt | 独立 .md 文件 | 配置与代码分离，易于修改 |
| 安全策略 | LLM 语义约束 + 代码拦截 | 纵深防御，策略表达 > 规则匹配 |
