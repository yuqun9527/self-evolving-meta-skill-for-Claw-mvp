# claw-mvp 项目概览

## 项目定位

claw-mvp 是 OpenClaw 的 MVP 轻量版：单进程 CLI AI Agent，同时支持 Web 聊天界面。TypeScript (ESM)，Node.js 22.6+ 用 `--experimental-strip-types` 直接运行，无需编译。唯一运行时依赖：`openai`。

## 运行方式

### CLI 模式
```bash
npm start
# 等价于: node --experimental-strip-types src/index.ts
```

### Web 模式
```bash
npm run web
# 等价于: node --experimental-strip-types src/server.ts
# 自定义端口: npm run web -- --port 8080
```

## 工具能力总览

| 工具 | 文件 | 功能 | 支持格式 |
|------|------|------|----------|
| **read** | `tools/read.ts` | 文件读取 | `.txt` `.md` `.csv` `.json` `.html` `.xml` `.yaml` `.log` `.py` `.ts` `.js` **`.xlsx` `.docx` `.pptx` `.pdf`** |
| **write** | `tools/write.ts` | 文件写入/创建 | 纯文本 + **`.xlsx` `.docx` `.pptx`**（自动转换） |
| **edit** | `tools/edit.ts` | 精确文本编辑 | 纯文本格式（oldText→newText 精确替换） |
| **exec** | `tools/exec.ts` | Shell 命令执行 | 跨平台（Win: PowerShell, 其他: bash） |
| **web_search** | `tools/web_search.ts` | 三引擎并行搜索 | Bing + Google + 百度，结果合并去重，无需 API Key |
| **web_fetch** | `tools/web_fetch.ts` | 网页抓取 | HTTP/HTTPS → 纯文本提取 |
| **download** | `tools/download.ts` | 文件下载 | PDF、图片等任意类型，流式写入工作区 |
| **python** | `tools/python.ts` | Python 脚本执行 | 任意 Python 脚本，自动 pip 安装依赖 |

## Web 服务

| 模块 | 文件 | 功能 |
|------|------|------|
| **Web 服务** | `server.ts` | 原生 `node:http` + SSE 流式聊天 |
| **前端界面** | `public/index.html` | 暗色主题 SPA 聊天界面，流式显示 + 工具调用可视化 |

**Web 模式特性**：
- REST API：`POST /api/chat`（SSE 流式）、`GET/POST/DELETE /api/sessions`、`GET /api/config`、`GET/POST /api/models`
- SSE 事件类型：`session` / `text` / `tool_call` / `tool_result` / `error` / `done`
- 多会话管理：内存 Map 按 `sessionId` 隔离 Agent 实例
- 端口可配：CLI `--port` > 环境变量 `CLAW_PORT` > 默认 3456

## Office/PDF 支持详情

**read 工具**：自动检测 `.xlsx`/`.docx`/`.pptx`/`.pdf` 扩展名，通过 Python 子进程提取文本：
- Excel → 遍历所有 Sheet，`\t` 分隔列
- Word → 提取段落 + 表格
- PPT → 按 Slide 提取文本内容
- PDF → 按 Page 提取（优先 pdfplumber，备用 PyPDF2）

**write 工具**：自动检测目标扩展名，通过 Python 转换写入：
- `.xlsx` → CSV/TSV 内容自动转 Excel 电子表格
- `.docx` → Markdown 内容自动转 Word 文档（`#` 转标题）
- `.pptx` → `# 标题` 行 = 新幻灯片，`- 项目` = 列表项

**python 工具**：通用 Python 执行能力，用于复杂 Office 操作和数据分析：
- 自动 pip install 缺失的核心包（openpyxl、python-docx、python-pptx、pdfplumber、PyPDF2）
- 通过 `packages` 参数按需安装额外包（如 pandas、matplotlib）
- 默认 60s 超时，最大 300s
- 设置 `PYTHONIOENCODING=utf-8` 和 `PYTHONUTF8=1` 解决 Windows GBK 编码问题
- 临时脚本写入 workspace，执行后自动清理
