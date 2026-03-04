# OpenClaw 鉴权与运行时安全边界设计串讲

## 1. 鉴权模式设计

Gateway 支持 token/password/trusted-proxy 等模式，并有模式来源判定：

- override
- config
- env 自动推导
- default

关键地面：

- `src/gateway/auth.ts`
- `src/gateway/startup-auth.ts`

这使“运行时临时覆盖”和“持久配置”不会混淆。

## 2. 启动期鉴权补齐

`ensureGatewayStartupAuth` 会在 token 模式缺失 token 时生成随机 token，并根据策略决定是否持久化。

关键地面：

- `src/gateway/startup-auth.ts`

安全要点：

1. override 模式下默认不落盘，避免隐式改写长期策略。
2. hooks token 与 gateway token 禁止相同，避免入口混淆。

## 3. Tailscale 信任链

在 `allowTailscale` 场景，系统不仅看头，还会验证代理来源与 whois 信息一致性。

关键地面：

- `src/gateway/auth.ts`

这避免了伪造转发头导致的错误信任。

## 4. 安全边界不止鉴权

OpenClaw 运行时安全边界是多层的：

- 网关鉴权（谁能进）
- 方法级策略（能做什么）
- 沙箱工具策略（工具能碰什么）
- exec approval（高风险动作需人工确认）

关键地面：

- `src/gateway/server-methods/exec-approval.ts`
- `src/infra/exec-approvals.ts`
- `src/agents/sandbox/tool-policy.ts`

## 5. 配置热重载与安全一致性

配置修改后，Gateway 不会盲目全量重启，而是通过 reload plan 判断：

- hot reload
- restart
- noop

关键地面：

- `src/gateway/config-reload.ts`
- `src/gateway/server-reload-handlers.ts`

价值：减少抖动，同时保持鉴权与策略变更可控生效。

## 6. 一个真实动作链

管理员执行：

"切换到 password 模式并禁用 tailscale 免密访问。"

系统动作：

1. 配置写入后触发 reload 计划评估。
2. Gateway 更新 auth 解析结果并刷新相关 handler。
3. 新连接按新策略握手；旧连接按会话生命周期自然退出或被替换。
4. 审计日志与状态输出可见策略已收敛。
