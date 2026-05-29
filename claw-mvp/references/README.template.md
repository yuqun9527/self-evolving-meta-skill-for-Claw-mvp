# claw-mvp — 最小的能安全办公的office claw

> CLI + Web AI Agent 轻量版 — 单进程 TypeScript (ESM)，Node.js 直接运行，零编译。

claw-mvp 是 OpenClaw 的 MVP 精简实现，支持 **CLI 交互模式** 和 **Web 聊天界面** 两种方式使用。基于 DeepSeek V4，具备工具调用、流式输出、会话管理等能力。

### 核心主打

**轻量** — 零编译、单依赖（仅 `openai`），16 个源文件，Node.js 直跑。没有 webpack / vite / tsc，clone 下来 `npm install && npm start` 就能用。

**安全** — 工作区沙箱隔离 + exec 危险命令黑名单 + 文件操作沙箱检测 + SSRF 内网 IP 拦截 + 反 LLM 幻觉规则。数据不足时诚实报告，不编造内容。

**办公** — 内置 `.xlsx`、`.docx`、`.pptx`、`.pdf` 自动解析与生成，Python 脚本执行 + 自动 pip install。三引擎并行搜索（Bing + Google + 百度），PDF 一键下载。真正能干活的工作 Agent。

---

## 📋 目录

- [快速开始](#快速开始)
- [运行方式](#运行方式)
  - [CLI 模式](#cli-模式)
  - [Web 模式](#web-模式)
- [斜杠命令](#斜杠命令)
- [Web API](#web-api)
- [工具能力](#工具能力)
- [配置说明](#配置说明)
- [文件结构](#文件结构)

---

## 快速开始

### 环境要求

- Node.js ≥ 22.6（需要 `--experimental-strip-types`）
- DeepSeek API Key

### 安装

```bash
cd claw-mvp
npm install
```

### 配置 API Key

编辑 `~/.claw/config.json`：

```json
{
  "model": "deepseek/deepseek-v4-flash",
  "providers": {
    "deepseek": {
      "baseURL": "https://api.deepseek.com/v1",
      "apiKey": "sk-你的API密钥"
    }
  },
  "workspace": "./workspace",
  "maxHistoryTurns": 50,
  "maxIterations": 30,
  "maxSearchRounds": 4,
  "proxy": "http://127.0.0.1:7890"
}
```

也可以使用环境变量：

```bash
set DEEPSEEK_API_KEY=sk-你的API密钥    # Windows
export DEEPSEEK_API_KEY=sk-你的API密钥  # Linux/Mac
```

---

## 运行方式

### CLI 模式

```bash
npm start
```

启动后进入 REPL 交互界面：

```
╔══════════════════════════════════════════════════╗
║  claw v0.2.0 │ CLI AI Agent                     ║
║  Type your message or /help for commands        ║
║  Model: deepseek/deepseek-v4-flash              ║
╚══════════════════════════════════════════════════╝

gr> 你好，帮我总结一下这份文件
```

- 输入自然语言消息与 AI 对话
- AI 会自动调用工具（读文件、搜索、执行命令等）完成任务
- 支持多轮工具调用，流式输出

### Web 模式

```bash
npm run web
```

浏览器打开 **http://localhost:3456**

| 特性 | 说明 |
|------|------|
| 暗色主题 | 类似 OpenClaw 的聊天界面风格 |
| 流式输出 | SSE 实时显示 AI 回复内容 |
| 工具调用可视 | 绿色标签显示工具名称和参数 |
| 会话管理 | URL 参数 `?session=xxx` 标识会话，支持多会话 |
| 新建/清空 | 界面按钮一键操作 |

#### 自定义端口

三种方式，优先级从高到低：

```bash
# 方式一：命令行参数（推荐）
npm run web -- --port 8080
npm run web -- -p 8080
npm run web -- 8080              # 直接跟端口号也可以

# 方式二：环境变量（Windows）
set CLAW_PORT=8080 && npm run web

# 方式三：环境变量（Linux / Mac）
CLAW_PORT=8080 npm run web
```

默认端口：**3456**

---

## 斜杠命令

CLI 模式下支持以下命令（在 `gr>` 提示符后输入）：

| 命令 | 说明 | 示例 |
|------|------|------|
| `/file <path>` | 注入文件内容到上下文 | `/file readme.txt` |
| `/model <name>` | 切换模型 | `/model deepseek/deepseek-chat` |
| `/clear` | 清屏 + 清空消息历史 | `/clear` |
| `/history` | 显示消息历史摘要 | `/history` |
| `/save [name]` | 保存当前会话 | `/save my-chat` |
| `/load <name>` | 加载已保存的会话 | `/load my-chat` |
| `/list` | 列出所有已保存会话 | `/list` |
| `/delete <name>` | 删除指定会话 | `/delete my-chat` |
| `/exit` | 退出程序 | `/exit` |
| `/help` | 显示帮助信息 | `/help` |

### 快捷键

| 按键 | 说明 |
|------|------|
| `Ctrl+C` | 中断当前 AI 生成 |
| `Ctrl+C` × 2 | 退出程序 |
| `Tab` | 自动补全命令 |

---

## Web API

Web 模式下提供的 REST API：

### `POST /api/chat`

发送消息，SSE 流式返回。

**请求：**
```json
{
  "message": "你好，介绍一下自己",
  "sessionId": "session_xxx"
}
```
- `sessionId` 可选，不传则使用 `default` 会话

**响应（SSE 事件流）：**

| 事件类型 | 数据字段 | 说明 |
|----------|----------|------|
| `session` | `{ sessionId, model, workspace }` | 会话信息 |
| `text` | `{ content }` | 流式文本块 |
| `tool_call` | `{ name, args }` | 工具调用 |
| `tool_result` | `{ name, result }` | 工具执行结果 |
| `error` | `{ message }` | 错误消息 |
| `done` | `{ sessionId }` | 请求完成 |

### `GET /api/sessions`

列出所有活跃会话。

### `POST /api/sessions`

创建新会话，返回 `{ sessionId, model }`。

### `DELETE /api/sessions/:id`

删除指定会话。

### `GET /api/config`

获取当前配置（模型、工作区等）。

---

## 工具能力

AI 可以自主选择并调用以下工具完成任务：

| 工具 | 功能 | 支持格式 |
|------|------|----------|
| **read** | 文件读取 | `.txt` `.md` `.csv` `.json` `.html` `.xml` `.yaml` `.log` `.py` `.ts` `.js` `.xlsx` `.docx` `.pptx` `.pdf` |
| **write** | 文件写入 | 纯文本 + `.xlsx` `.docx` `.pptx`（自动转换） |
| **edit** | 精确文本编辑 | 文本替换 / 行号替换 / 正则替换 |
| **exec** | Shell 命令执行 | 跨平台（Win: PowerShell / cmd, 其他: bash） |
| **web_search** | 三引擎并行搜索 | Bing + Google + 百度，结果合并去重，无需 API Key |
| **web_fetch** | 网页抓取 | HTTP/HTTPS → 文本提取 |
| **download** | 文件下载 | PDF、图片等任意类型，流式写入工作区 |
| **python** | Python 脚本执行 | 自动 pip 安装依赖，Office/数据分析 |

### 安全限制

- 所有文件操作限制在 workspace 目录内
- 危险命令黑名单拦截（rm -rf /、format 等）
- Python 脚本代码安全检查（禁用 os.system、subprocess 等）
- 搜索轮次上限可配置（默认 4 轮，`maxSearchRounds`），防止无限搜索循环
- 反空话检测：防止 AI 只说不做

---

## 配置说明

完整配置项（`~/.claw/config.json`）：

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `model` | string | `deepseek/deepseek-chat` | 模型标识（provider/model） |
| `providers` | object | `{ deepseek: {...} }` | LLM 服务商配置 |
| `providers.xxx.baseURL` | string | `https://api.deepseek.com/v1` | API 地址 |
| `providers.xxx.apiKey` | string | `""` | API 密钥 |
| `workspace` | string | `./workspace` | 工作区目录 |
| `maxHistoryTurns` | number | `50` | 最大保留消息轮数 |
| `maxIterations` | number | `30` | 单次任务最大工具调用轮数 |
| `maxSearchRounds` | number | `4` | 搜索轮次上限 |
| `proxy` | string | `"http://127.0.0.1:7890"` | HTTP/HTTPS 代理地址 |
| `systemPrompt` | string | 自动加载 | 系统提示词（优先 system-prompt.md） |

### System Prompt

加载优先级：
1. `config.json` 中的 `systemPrompt` 字段
2. 项目根目录的 `system-prompt.md`
3. 内置默认提示词

---

## 文件结构

```
claw-mvp/
├── package.json          # 项目配置与脚本
├── package-lock.json     # 依赖锁定文件
├── node_modules/         # 依赖包（npm install 自动生成）
├── system-prompt.md      # 默认 System Prompt
├── README.md             # 本文件
├── public/               # Web 静态资源
│   └── index.html        # 聊天界面（SPA）
├── src/                  # 源代码
│   ├── index.ts          # CLI 入口（REPL 循环 + 斜杠命令）
│   ├── server.ts         # Web 服务入口（HTTP + SSE 流式 API）
│   ├── agent.ts          # Agent 主循环（流式处理 + 工具调用编排）
│   ├── llm.ts            # LLM 调用封装（OpenAI SDK + DeepSeek V4）
│   ├── config.ts         # 配置加载（config.json + 环境变量）
│   ├── session.ts        # 会话持久化（保存/加载/列表/删除）
│   └── tools/            # 工具实现
│       ├── index.ts      # 工具注册表
│       ├── read.ts       # 文件读取（含 Office/PDF）
│       ├── write.ts      # 文件写入（含 Office 转换）
│       ├── edit.ts       # 文本编辑
│       ├── exec.ts       # Shell 执行
│       ├── web_search.ts # 网页搜索
│       ├── web_fetch.ts  # 网页抓取
│       ├── download.ts  # 文件下载
│       └── python.ts     # Python 脚本执行
└── workspace/            # 默认工作区（AI 操作文件存放处，gitignore）
```

---

## 技术栈

- **运行时**: Node.js 22.6+ `--experimental-strip-types`
- **语言**: TypeScript (ESM)，零编译
- **LLM**: OpenAI SDK + DeepSeek V4（支持 reasoning_content）
- **Web**: 原生 `node:http` + SSE（Server-Sent Events）
- **依赖**: `openai`（唯一运行时依赖）

---

## 开发

```bash
# 类型检查
npm run check

# CLI 模式
npm start

# Web 模式（默认端口 3456）
npm run web
npm run web -- --port 8080

# 运行测试（如适用）
node --experimental-strip-types test-harness.ts
```
