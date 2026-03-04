# OpenClaw 插件系统与 Skills 系统串讲

## 目标

本文拆开讲两条扩展线：

- 插件系统：扩展渠道、网关方法、工具、HTTP 路由。
- Skills 系统：扩展 agent 的可执行知识与操作手册。

## 1. 插件系统设计

插件注册中心聚合以下能力：

- tools
- channels
- providers
- gateway methods
- hooks
- cli commands
- services
- http handlers

关键地面：

- `src/plugins/registry.ts`
- `src/plugins/loader.ts`
- `src/gateway/server-plugins.ts`

核心特点：

1. 统一注册接口。
2. 冲突检测（如 method 重名）。
3. 诊断信息上报（warn/error）。
4. Gateway 启动阶段自动合并能力面。

## 2. 渠道插件如何接入主链路

渠道插件通过 registry 注入后，Gateway 在运行时统一发现与调度。

关键地面：

- `src/channels/plugins/index.ts`
- `src/gateway/server-channels.ts`
- `src/gateway/server-methods-list.ts`

结果是：新增渠道通常不需要改核心分发逻辑，主要实现插件协议即可。

## 3. Skills 系统设计

Skills 提供给 agent 的是“结构化能力包”，包括说明、触发条件、执行约束和相关命令。

关键地面：

- `src/agents/skills/workspace.ts`
- `src/agents/skills/frontmatter.ts`
- `src/agents/skills/config.ts`
- `src/agents/skills/filter.ts`

主要能力：

1. 多来源加载（bundled、workspace、managed、plugin）。
2. frontmatter 解析与调用策略。
3. 技能过滤与序列化。
4. prompt 注入时的体积控制和路径压缩。

## 4. 插件与 Skills 的关系

二者关系是互补，不是替代：

- 插件偏系统扩展面（接口、协议、运行时能力）。
- Skills 偏 agent 认知与执行面（如何做事、按什么流程做）。

关键地面：

- `src/agents/skills/plugin-skills.ts`
- `src/infra/skills-remote.ts`
- `src/agents/skills/refresh.ts`

## 5. 实战链路示例

需求：

"新增一个企业内部渠道，并提供一套迁移评估 skill。"

系统动作：

1. 通过插件新增 channel + gateway methods + outbound 适配。
2. 通过 Skills 新增迁移评估流程模板。
3. Gateway 启动时加载插件能力。
4. Agent run 时按过滤规则注入目标 skill。
5. 用户在新渠道发起任务，得到按 skill 模板约束的输出。
