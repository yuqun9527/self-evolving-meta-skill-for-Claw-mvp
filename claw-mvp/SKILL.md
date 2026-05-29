---
name: claw-mvp
description: "OpenClaw 轻量版 CLI AI Agent（claw-mvp）的开发与维护。Trigger：用户要求开发/修改/扩展 claw-mvp 项目，添加新工具/命令/功能到 claw-mvp，修复 claw-mvp Bug，理解 claw-mvp 代码架构。"
---

# claw-mvp 开发指南

claw-mvp 是 OpenClaw 的 MVP 轻量版：单进程 CLI + Web 双模式 AI Agent。TypeScript (ESM)，Node.js 22.6+ 用 `--experimental-strip-types` 直接运行，零编译。唯一依赖：`openai`。

---

## 📖 按需索引

根据当前任务需求，只读取对应的子文档（`references/` 目录）：

| 需求场景 | 读取文件 |
|----------|----------|
| 了解项目能力、工具列表、运行方式 | [overview.md](references/overview.md) |
| 了解开发流程（设计→开发→测试三阶段） | [workflow.md](references/workflow.md) |
| 查看编码铁律和注意事项 | [dev-notes.md](references/dev-notes.md) |
| 从头开发，逐项检查不踩坑 | [checklist.md](references/checklist.md) |
| 深度理解架构和模块设计 | [design-doc.md](references/design-doc.md) |
| 查看代码实现模式和模板 | [code-patterns.md](references/code-patterns.md) |
| 排查 Bug、了解已知问题 | [bug-tracker.md](references/bug-tracker.md) |
| 安全测试用例（攻击载荷）| [security-test-cases.md](references/security-test-cases.md) |
| 生成项目 README.md | [README.template.md](references/README.template.md) |
| System Prompt 模板 | [system-prompt.md](references/system-prompt.md) |
| Web 前端界面模板（⚠️ 严格参考）| [web/index.html](references/web/index.html) |
| 多 Agent 协作开发流程 | [../dev-team-workflow/SKILL.md](../dev-team-workflow/SKILL.md) |

> ⚠️ **Web 界面铁律**：后续所有 claw-mvp 的 Web 界面必须以 `references/web/index.html`（技能自带模板）为严格参考模板。包括但不限于：深蓝紫主题、可折叠工具步骤卡片、会话管理、模型切换、SSE 解析方式（`\n\n` 分割）、`hasToolCalls` 复位逻辑。不得自行设计新的视觉风格或交互模式。详见 [dev-notes.md](references/dev-notes.md) § Web 前端。

## 🗂 文档结构

```
skills/claw-mvp/
├── SKILL.md                       # ← 本文件（元数据 + 索引）
└── references/
    ├── overview.md                # 项目概览（工具能力、Web服务、运行方式）
    ├── workflow.md                # 开发工作流（三阶段 + sub-agent配置）
    ├── dev-notes.md               # 开发注意事项（27条铁律）
    ├── checklist.md               # 从零开发检查清单（编码/安全/测试/验证）
    ├── design-doc.md              # 完整设计文档（架构/接口/安全/决策记录）
    ├── code-patterns.md           # 代码模式（工具模板/Agent Loop/Web服务）
    ├── bug-tracker.md             # Bug追踪 + 教训库 + 常见问题
    ├── security-test-cases.md     # 安全测试用例集（攻击载荷）
    ├── system-prompt.md           # System Prompt 完整模板
    └── README.template.md         # README 结构参考

skills/dev-team-workflow/
    └── SKILL.md                   # 多Agent协作开发流程
```

## 🚀 快速开始

```bash
# 从零开发 claw-mvp 时的推荐读取顺序：
# 1. overview.md     — 知道要做什么
# 2. workflow.md     — 知道怎么做
# 3. checklist.md    — 逐项核对不踩坑
# 4. 如遇问题 → dev-notes.md / bug-tracker.md / code-patterns.md
# 5. 如做架构决策 → design-doc.md
```
