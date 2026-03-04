# OpenClaw 模型调用 上下文管理 安全沙箱 串讲

## 目标

本文聚焦一次 agent run 的核心执行面：

- 模型如何被选择与回退。
- 上下文如何构建和守护。
- 工具调用如何被沙箱与策略约束。

## 1. 模型调用入口

`agentCommand` 会根据 provider 类型选择执行路径：

- CLI provider 走 `runCliAgent`
- 嵌入式 provider 走 `runEmbeddedPiAgent`

关键地面：

- `src/commands/agent.ts`
- `src/agents/pi-embedded.ts`
- `src/agents/pi-embedded-runner.ts`

## 2. 模型选择与回退

OpenClaw 会综合配置、会话、auth profile 与失败原因做 failover。

关键地面：

- `src/agents/model-selection.ts`
- `src/agents/model-fallback.ts`
- `src/agents/pi-embedded-runner/run.ts`

机制要点：

1. 先选主模型。
2. 检查上下文窗口下限。
3. 调用失败后按策略切换候选模型或 auth profile。
4. 记录失败原因并反馈可读错误。

## 3. 上下文管理

上下文不是“全量无脑拼接”，而是有预算和守护：

- history turns 限制
- token 预算校验
- 过载时触发 compaction
- tool 结果截断与保护

关键地面：

- `src/agents/pi-embedded-runner/history.ts`
- `src/agents/pi-embedded-runner/compact.ts`
- `src/agents/pi-embedded-runner/tool-result-truncation.ts`
- `src/agents/context-window-guard.ts`

## 4. 安全沙箱

OpenClaw 的工具执行不是裸跑，沙箱层负责范围和风险约束。

关键地面：

- `src/agents/sandbox.ts`
- `src/agents/sandbox/context.ts`
- `src/agents/sandbox/tool-policy.ts`
- `src/agents/sandbox/validate-sandbox-security.ts`

策略要点：

1. 按 agent/global 配置解析 allow/deny。
2. 工具组展开后做 glob 匹配。
3. 默认拒绝高风险工具集合。
4. 对会话工作目录和挂载路径做边界控制。

## 5. exec approval 人机协作闸门

对于高风险执行（如系统命令），还可要求显式审批。

关键地面：

- `src/gateway/exec-approval-manager.ts`
- `src/gateway/server-methods/exec-approval.ts`
- `src/infra/exec-approvals.ts`
- `src/agents/bash-tools.exec-approval-request.ts`

## 6. 实战链路示例

用户请求：

"帮我跑一段迁移脚本，并把结果汇总发给我。"

系统动作：

1. `agentCommand` 发起 run，解析模型和会话上下文。
2. 上下文窗口检查，必要时压缩历史。
3. 模型尝试执行工具调用。
4. 命中高风险命令时触发 exec approval。
5. 审批通过后继续执行，否则返回阻断说明。
6. 输出结果并写回会话。
