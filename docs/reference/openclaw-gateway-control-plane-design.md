# OpenClaw Gateway 控制平面设计串讲

## 1. 为什么 Gateway 是核心控制面

在 OpenClaw 中，Gateway 不是单纯 API 网关，而是统一的控制平面：

- 接收多端客户端（CLI、WebChat、桌面端、移动端、节点）的请求。
- 暴露统一 RPC 方法和事件流。
- 编排渠道生命周期、插件能力、节点配对、会话服务。

关键地面：

- `src/gateway/server.impl.ts`
- `src/gateway/server-runtime-state.ts`
- `src/gateway/server-methods-list.ts`

## 2. 启动编排分层

`startGatewayServer` 启动阶段做的是“分层组装”，不是单体初始化：

1. 配置迁移与合法性校验。
2. 启动鉴权补齐（必要时生成 token）。
3. 插件与渠道能力加载。
4. 解析运行时配置（bind/auth/tailscale/UI endpoint）。
5. 构建 HTTP + WS 运行时状态。
6. 挂载方法处理器和事件广播。

关键地面：

- `src/gateway/server.impl.ts`
- `src/gateway/startup-auth.ts`
- `src/gateway/server-runtime-config.ts`

## 3. 运行时状态对象的职责

`createGatewayRuntimeState` 负责组装最核心 runtime 实体：

- HTTP server（可多 bind host）
- WebSocket server
- client 集合与广播器
- chat run 状态、dedupe、abort 控制器
- hooks/plugin HTTP handler

关键地面：

- `src/gateway/server-runtime-state.ts`
- `src/gateway/server-broadcast.ts`
- `src/gateway/server-http.ts`

设计价值：把网络层、会话层、插件入口层统一收敛，便于后续热重载与停机编排。

## 4. WS 连接与消息处理链

WS 链路分两层：

1. 连接层（握手、鉴权、presence、元信息）。
2. 消息层（req 分发、方法执行、统一响应、错误包装）。

关键地面：

- `src/gateway/server/ws-connection.ts`
- `src/gateway/server/ws-connection/message-handler.ts`
- `src/gateway/server/ws-types.ts`

这让“连接管理”与“方法执行”分离，降低协议演进风险。

## 5. 方法面与事件面

Gateway 的能力面由两部分组成：

- 方法集合：基础方法 + 插件扩展方法。
- 事件集合：agent/chat/health/presence/cron/node/approval 等。

关键地面：

- `src/gateway/server-methods-list.ts`
- `src/gateway/server-methods.ts`
- `src/gateway/events.ts`

## 6. 一个真实动作链

用户在 WebChat 提交：

"继续昨天的调研，把第 2 个项目风险细化成三级。"

Gateway 动作：

1. WS 收到请求，完成鉴权。
2. message handler 将 `agent` 请求交给对应 handler。
3. handler 解析会话与路由，触发 agent 执行。
4. 过程事件（如 chat delta/presence）经广播器推给订阅客户端。
5. 最终响应通过同一 conn 返回，并同步更新会话状态。
