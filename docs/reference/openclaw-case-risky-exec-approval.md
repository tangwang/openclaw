# OpenClaw 典型案例 高风险命令执行与审批闭环剖析

## 场景

用户请求：

"请直接执行这段部署脚本并重启服务。"

该请求可能触发高风险工具调用（系统命令、文件改写、网络副作用），需要审批闭环。

## 1. 风险识别入口

Agent 尝试调用相关工具时，工具策略层先判断是否属于高风险动作。

关键地面：

- `src/security/dangerous-tools.ts`
- `src/agents/tool-policy.ts`
- `src/agents/sandbox/tool-policy.ts`

系统动作：

1. 检查工具是否在 deny 列表或超出 allow 范围。
2. 判断是否需要 exec approval。

## 2. 沙箱边界先行

即使审批通过，执行也不应越过沙箱边界（工作目录、挂载范围、环境变量）。

关键地面：

- `src/agents/sandbox/context.ts`
- `src/agents/sandbox/validate-sandbox-security.ts`
- `src/agents/sandbox/fs-paths.ts`

## 3. 审批请求与等待决策

命中审批路径后，Gateway 会创建 approval request 并等待批准/拒绝。

关键地面：

- `src/gateway/exec-approval-manager.ts`
- `src/gateway/server-methods/exec-approval.ts`
- `src/agents/bash-tools.exec-approval-request.ts`

系统动作：

1. 生成审批请求并广播到可见客户端。
2. 阻塞该高风险步骤（非整条会话硬中断）。
3. 收到决策后继续或回退。

## 4. 执行与审计

审批通过后，任务继续执行；拒绝则返回阻断说明，并保留审计痕迹。

关键地面：

- `src/infra/exec-approvals.ts`
- `src/infra/exec-approval-forwarder.ts`
- `src/gateway/server-methods/exec-approvals.ts`

## 5. 失败与恢复路径

常见失败路径：

- 审批超时
- 客户端断连
- 命令运行中止

系统通过 queue 与 run 状态管理保证失败可感知、可恢复、可追踪。

关键地面：

- `src/process/command-queue.ts`
- `src/agents/pi-embedded-runner/abort.ts`
- `src/gateway/server-close.ts`

## 6. 机制总结

这个案例体现了 OpenClaw 的安全执行闭环：

- 前置策略识别风险。
- 沙箱限制可操作边界。
- 审批系统做人机闸门。
- 执行结果进入审计与会话沉淀。
