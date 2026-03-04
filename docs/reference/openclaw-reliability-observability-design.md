# OpenClaw 可靠性与可观测性设计串讲

## 1. 可靠性目标

OpenClaw 面向长时运行场景，重点不是“单次成功”，而是：

- 渠道异常可恢复
- 重启可控
- 队列不丢语义
- 关键状态可观测

## 2. 渠道自愈

渠道生命周期由 channel manager 和 health monitor 协作：

- 渠道崩溃后指数退避重启
- 手动停止后不自动拉起
- 健康探测触发限流重启

关键地面：

- `src/gateway/server-channels.ts`
- `src/gateway/channel-health-monitor.ts`

## 3. 重启延迟与队列保护

Gateway 重启不是立即硬切，而是结合待处理队列与待发回复做 deferral。

关键地面：

- `src/infra/restart.ts`
- `src/process/command-queue.ts`
- `src/gateway/server-restart-deferral.test.ts`

目的：减少“正在执行中的任务”被粗暴打断。

## 4. 心跳与维护任务

系统通过 heartbeat runner 和 maintenance timers 做周期治理：

- 心跳
- session 维护
- 状态广播与健康快照刷新

关键地面：

- `src/infra/heartbeat-runner.ts`
- `src/gateway/server-maintenance.ts`
- `src/gateway/server/health-state.ts`

## 5. 可观测性体系

OpenClaw 可观测性分三层：

1. 结构化日志与子系统 logger。
2. WS 连接和消息日志（含握手/帧维度）。
3. 诊断心跳与会话状态快照。

关键地面：

- `src/logging/subsystem.ts`
- `src/gateway/ws-log.ts`
- `src/logging/diagnostic.ts`

## 6. 一个真实动作链

凌晨某渠道连接反复断开，系统动作：

1. channel runtime 记录错误并标记 stopped。
2. health monitor 观察到异常，按冷却策略执行重启。
3. 若达到每小时重启上限，则进入保护并打告警日志。
4. 运维通过 status/health 与 ws-log 观察恢复进展。

这样可以把“偶发抖动”限制为可管理事件，而不是全局故障。
