# OpenClaw 消息路由设计串讲

## 目标

本文专门讲清楚 OpenClaw 的消息路由链路：

- 消息从渠道进入后如何归一化。
- 如何根据绑定规则选中 agent。
- 如何生成并复用 sessionKey。
- 如何把结果可靠送回原渠道。

## 1. 入站统一到 Gateway agent 方法

各渠道先把原始消息转换成统一请求，最终进入 Gateway 的 `agent` 方法。

关键地面：

- `src/gateway/server-methods/agent.ts`
- `src/gateway/server-methods-list.ts`

典型处理顺序：

1. 参数校验（消息、渠道、附件、sessionKey 等）。
2. 幂等去重（`idempotencyKey`）。
3. 附件归一化（文本 + 图片等统一结构）。
4. 进入 `agentCommand` 执行。

## 2. 路由核心 resolve-route

路由逻辑在 `resolve-route.ts`，它把消息上下文映射为：

- `agentId`
- `sessionKey`
- `mainSessionKey`
- `matchedBy`（命中路径）

关键地面：

- `src/routing/resolve-route.ts`
- `src/routing/bindings.ts`

命中优先级要点：

1. peer 精确绑定
2. parent peer 继承
3. guild + roles
4. guild/team/account/channel
5. 默认 agent

这样可以保证“同一人同一会话”的连续性，同时支持群聊和组织级路由规则。

## 3. sessionKey 设计

OpenClaw 不是只靠 channel id 做会话，而是用结构化 `sessionKey` 表达 agent + 频道 + 账户 + peer 语义。

关键地面：

- `src/routing/session-key.ts`
- `src/routing/resolve-route.ts`
- `src/config/sessions.ts`

价值：

- 避免不同渠道/账号串会话。
- 支持 DM 聚合策略（main/per-peer/per-channel-peer 等）。
- 支持后续查询、压缩、投递策略的统一主键。

## 4. 目标解析与回传

当 agent 产出回复后，系统走 outbound 抽象层回传：

1. 先尝试渠道插件动作。
2. 插件未处理时回退核心发送逻辑。
3. 成功后可镜像写回 transcript。

关键地面：

- `src/infra/outbound/outbound-send-service.ts`
- `src/infra/outbound/target-resolver.ts`
- `src/channels/plugins/index.ts`

这使不同渠道的差异（thread id、mention 格式、目标标识）被插件吸收，核心流保持统一。

## 5. 实战链路示例

用户在 Discord 发出：

"帮我继续昨天那个迁移方案，先比较 A/B 两个仓库的风险。"

系统动作：

1. Discord 插件把消息标准化并上报 Gateway。
2. `resolve-route` 命中该用户历史绑定，选中目标 agent。
3. 用既有 `sessionKey` 读取上下文，避免重复解释背景。
4. Agent 执行并得到结果。
5. outbound 选择 Discord 通道适配器发回线程。
6. transcript 更新，下一轮可直接“继续第 2 点”。

## 6. 排障观测点

遇到“回错会话”或“发错目标”时，优先看：

- `matchedBy` 命中类型是否符合预期。
- sessionKey 形态是否正确。
- 目标解析是否命中歧义目录项。
- outbound 是否走了插件路径还是核心回退路径。

关键地面：

- `src/routing/resolve-route.test.ts`
- `src/routing/session-key.test.ts`
- `src/infra/outbound/target-resolver.test.ts`
