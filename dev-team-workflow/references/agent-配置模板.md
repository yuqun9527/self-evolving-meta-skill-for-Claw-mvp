# OpenClaw 多Agent配置模板

## openclaw.json 配置

为每个角色在 `agents.list` 中注册：

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: { primary: "deepseek/deepseek-v4-flash" },
      subagents: {
        allowAgents: ["designer", "developer", "tester"],
        maxSpawnDepth: 3,        // 允许嵌套（如 tester spawn 子任务）
        maxConcurrent: 5,         // 最多并行子任务
        runTimeoutSeconds: 600,   // 单任务超时
        archiveAfterMinutes: 120, // 完成后自动归档
      },
    },
    list: [
      {
        id: "main",  // 项目经理（默认 Agent）
        subagents: { allowAgents: ["designer", "developer", "tester"] },
      },
      {
        id: "designer",
        name: "Designer",
        workspace: "~/.openclaw/workspace-designer",
        agentDir: "~/.openclaw/agents/designer/agent",
        subagents: { allowAgents: [] },  // 不派活
      },
      {
        id: "developer",
        name: "Developer",
        workspace: "~/.openclaw/workspace-developer",
        agentDir: "~/.openclaw/agents/developer/agent",
        tools: { profile: "coding" },    // 给满工具权限
        subagents: { allowAgents: [] },
      },
      {
        id: "tester",
        name: "Tester",
        workspace: "~/.openclaw/workspace-tester",
        agentDir: "~/.openclaw/agents/tester/agent",
        subagents: { allowAgents: [] },
      },
    ],
  },
}
```

## 创建Agent的命令

```bash
# 逐个创建
openclaw agents add designer
openclaw agents add developer
openclaw agents add tester

# 查看
openclaw agents list --bindings

# 重启后生效
openclaw gateway restart
```

## 各Agent的Workspace文件清单

```
~/.openclaw/workspace-<agentId>/
├── SOUL.md      # 角色定义（必写）
├── AGENTS.md    # 工作规则
├── IDENTITY.md  # 身份标识
├── TOOLS.md     # 工具说明
├── USER.md      # 用户信息
└── HEARTBEAT.md # 心跳（保持空内容跳过）
```