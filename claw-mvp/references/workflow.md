# claw-mvp 开发工作流

## 推荐流程：dev-team-workflow

> 详细流程见 [dev-team-workflow SKILL](../../dev-team-workflow/SKILL.md)

按 **设计⇄审查 + 开发⇄测试** 双迭代循环，spawn `designer` / `developer` / `tester` agent：

### 阶段一前置：Sub-agent 配置

sub-agent 必须用 `deepseek-v4-flash` + `thinking: "off"` + `runTimeoutSeconds: 300`。不能用 `deepseek-v4-pro`（22 分钟超时）。

spawn 参数模板：
```
sessions_spawn(
  agentId: "xxx",
  model: "deepseek/deepseek-v4-flash",
  thinking: "off",
  runTimeoutSeconds: 300,
  ...
)
```

### 阶段一：设计-审查循环

1. Designer 出方案设计文档（参考 [design-doc.md](design-doc.md)）
2. Developer 审查设计文档，有问题给出 List
3. Designer 按 List 修订 → Developer 再审 → 循环直到 Developer 确认无问题
4. Designer 输出到 `D:\openclaw\claw-mvp\01-设计文档.md`
5. Developer 审查输出到 `D:\openclaw\claw-mvp\01-审查意见.md`
6. 最多 5 轮，超限向用户报告分歧

### 阶段二：开发阶段

设计一致后，Developer 编码实现 + 自测。按 [checklist.md](checklist.md) 逐项核对。

### 阶段三：开发-测试循环

1. Tester 编写测试用例 + 执行测试，有问题给出 Bug List
2. Developer 按 List 修复 Bug → Tester 再测 → 循环直到 Tester 确认无问题
3. Tester 测试报告输出到 `D:\openclaw\claw-mvp\03-测试报告.md`
4. 覆盖所有工具（7个）、配置模块、会话模块、Web 服务
5. 最多 10 轮

## 小改动流程

简单 Bug 修复或小功能，直接用现有工具修改源码并验证运行，无需走完整三阶段。
