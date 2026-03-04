# OpenClaw 典型案例 GitHub 迁移可行性调研全链路剖析

## 场景

用户请求：

"帮我在 GitHub 搜索 xx 相关项目，筛选可迁移到 yy 的候选，逐个阅读并给出技术方案和关键代码。"

这个请求本质上是一个多阶段 agentic pipeline，而不是一次单轮问答。

## 1. 入站与请求规范化

消息先从具体渠道进入（Discord/Telegram/WebChat 等），被统一转换成 Gateway `agent` 请求。

关键地面：

- `src/gateway/server-methods/agent.ts`
- `src/gateway/server/ws-connection/message-handler.ts`

系统动作：

1. 校验参数和渠道合法性。
2. 用 `idempotencyKey` 去重。
3. 解析附件（如用户附了参考仓库链接或截图）。

## 2. 路由与会话绑定

系统根据 channel/account/peer/thread 上下文解析 `agentId + sessionKey`，保证任务落到正确 agent 的连续上下文里。

关键地面：

- `src/routing/resolve-route.ts`
- `src/routing/session-key.ts`
- `src/gateway/server-session-key.ts`

系统动作：

1. 命中 bindings 规则（peer/guild/roles 等）。
2. 生成或复用 sessionKey。
3. 让后续“继续第 2 个项目深挖”成为可追踪延续。

## 3. 任务分解与执行计划

Agent 不会直接给结论，而是先把请求拆成任务图：

- 检索策略（关键词扩展、语言/活跃度/许可证过滤）。
- 迁移维度（架构耦合、依赖可替代性、部署形态）。
- 深读清单（entrypoint、核心模块、存储/接口抽象、插件点）。

关键地面：

- `src/commands/agent.ts`
- `src/agents/pi-embedded-runner/run.ts`
- `src/agents/model-selection.ts`

## 4. 工具调用与证据采集

Agent 调用浏览器/HTTP/文件工具读取仓库页面与代码文件，构建“候选集 -> 入围集”。

关键地面：

- `src/browser/pw-tools-core.ts`
- `src/agents/pi-embedded-subscribe.handlers.tools.ts`
- `src/agents/tool-policy.ts`

系统动作：

1. 召回仓库基础元数据（stars、最近提交、主要语言）。
2. 粗筛后进入代码级深读。
3. 提取关键代码片段及其设计职责。

## 5. 上下文预算与压缩

当项目数量多、代码片段长时，系统会触发上下文治理：

- history 限制
- 工具结果截断
- compaction 压缩

关键地面：

- `src/agents/pi-embedded-runner/history.ts`
- `src/agents/pi-embedded-runner/tool-result-truncation.ts`
- `src/agents/pi-embedded-runner/compact.ts`

这样可避免模型因超窗而失稳。

## 6. 生成报告与回传

输出不是散文，而是结构化结论：

- 候选仓库对比矩阵。
- 每个项目的技术方案摘要。
- 关键代码位置 + 为什么关键。
- 迁移优先级和分阶段路径建议。

关键地面：

- `src/infra/outbound/outbound-send-service.ts`
- `src/infra/outbound/target-resolver.ts`
- `src/config/sessions.ts`

系统动作：

1. outbound 先尝试插件发送，失败回退核心发送。
2. 结果写回 transcript。
3. 后续用户可继续追问并复用同一上下文。

## 7. 机制总结

这个案例同时触发了 OpenClaw 的关键机制：

- Gateway 控制面编排。
- 路由和 session 主键一致性。
- 模型调用与上下文预算守护。
- 工具调用策略与安全边界。
- 统一 outbound + 会话沉淀。
