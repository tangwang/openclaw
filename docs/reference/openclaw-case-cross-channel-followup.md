# OpenClaw 典型案例 跨渠道接力追问内部机制剖析

## 场景

用户在手机 Telegram 发起调研，回家后在 WebChat 继续说：

"继续刚才那个结论，把风险拆成三层并给出落地清单。"

目标是同一会话连续，不因换端或换渠道丢上下文。

## 1. 为什么跨渠道会话容易错乱

如果系统只用“当前渠道会话 id”做主键，会出现：

- 同一用户在不同渠道断裂。
- 群聊 thread 和私聊上下文混淆。

OpenClaw 用结构化 sessionKey + 路由规则来解决。

关键地面：

- `src/routing/session-key.ts`
- `src/routing/resolve-route.ts`
- `src/sessions/send-policy.ts`

## 2. 第一跳 Telegram 发起

系统动作：

1. Telegram 消息归一化后进入 Gateway `agent`。
2. 路由层解析到目标 agent + sessionKey。
3. 结果发送后写入 transcript。

关键地面：

- `src/telegram/bot-message-context.ts`
- `src/gateway/server-methods/agent.ts`
- `src/config/sessions.ts`

## 3. 第二跳 WebChat 继续

用户在 WebChat 说“继续刚才那个”，系统会通过会话解析策略找到可延续上下文。

关键地面：

- `src/gateway/sessions-resolve.ts`
- `src/gateway/server-methods/chat.ts`
- `src/web/session.ts`

系统动作：

1. 识别当前客户端与可继承会话。
2. 绑定同一 agent 语义上下文。
3. 继续推理，而不是重开空白会话。

## 4. 回传策略与投递目标

跨渠道时，回复目标不一定等于发起端。系统会按 send policy、delivery target、thread 信息决定回传位置。

关键地面：

- `src/infra/outbound/agent-delivery.ts`
- `src/infra/outbound/channel-selection.ts`
- `src/infra/outbound/target-resolver.ts`

## 5. 可靠性细节

跨渠道长会话常见问题：

- 并发追问造成上下文冲突。
- 某端断连后状态不一致。

OpenClaw 通过 lane/queue/presence 机制缓解。

关键地面：

- `src/process/command-queue.ts`
- `src/gateway/server/presence-events.ts`
- `src/gateway/server-lanes.ts`

## 6. 机制总结

这个案例展示了 OpenClaw 在“多入口单语义”上的核心设计：

- 路由层决定语义归属。
- sessionKey 保持上下文连续。
- outbound 决定最佳回传位置。
- queue 和 presence 保障并发一致性。
