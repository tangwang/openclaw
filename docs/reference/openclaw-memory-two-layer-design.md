# OpenClaw 记忆系统设计串讲 两层记忆 索引 压缩

## 目标

本文把 OpenClaw 记忆系统拆成两层来讲：

- 长期语义记忆层（检索索引）。
- 会话上下文压缩层（短期对话记忆）。

## 1. 两层记忆架构

### 第一层 长期语义记忆

面向跨会话检索，来源包括工作区文档与会话文件，核心是向量加关键词混合检索。

关键地面：

- `src/memory/manager.ts`
- `src/memory/manager-search.ts`
- `src/memory/hybrid.ts`
- `src/memory/sqlite-vec.ts`

### 第二层 会话压缩记忆

面向当前会话上下文预算，通过 compaction 把冗长历史浓缩为可继续推理的摘要。

关键地面：

- `src/agents/pi-embedded-runner/compact.ts`
- `src/agents/pi-embedded-runner/history.ts`
- `src/agents/pi-embedded-runner/run/compaction-timeout.ts`

## 2. 索引设计

`MemoryIndexManager` 负责索引生命周期：

1. 初始化 embedding provider。
2. 同步文件与 session 源。
3. 建立向量表和 FTS 表。
4. 查询时做 hybrid merge（向量 + BM25）。

关键地面：

- `src/memory/manager.ts`
- `src/memory/embeddings.ts`
- `src/memory/query-expansion.ts`
- `src/memory/sync-index.ts`

## 3. 检索路径

一次 memory search 典型路径：

1. query 扩展关键词。
2. FTS 召回语义相关片段。
3. 向量检索召回语义近邻。
4. 混合排序和去重。
5. 返回片段与来源元信息。

关键地面：

- `src/memory/search-manager.ts`
- `src/memory/hybrid.ts`
- `src/memory/mmr.ts`

## 4. 压缩设计

当会话过长或工具输出过大时，系统触发压缩：

- 统计消息贡献度。
- 生成可继续推理的紧凑摘要。
- 保留必要事实与行动点。
- 避免重复压缩与失真扩散。

关键地面：

- `src/agents/pi-embedded-runner/compact.ts`
- `src/agents/pi-embedded-runner/tool-result-context-guard.ts`
- `src/agents/pi-embedded-runner/tool-result-truncation.ts`

## 5. 为什么要两层

- 仅靠长期记忆：当前对话会超窗。
- 仅靠会话压缩：跨会话知识不可检索。
- 两层结合：既能长期记住，又能在单次推理窗口内稳定运行。

## 6. 实战链路示例

用户说：

"延续上周迁移讨论，结合仓库里的新文档，给我新的切换计划。"

系统动作：

1. 从长期记忆层检索上周关键文档与历史结论。
2. 将检索片段注入当前 run。
3. 若上下文过大，触发会话压缩层收敛历史。
4. 在可控 token 预算内生成新版计划。
