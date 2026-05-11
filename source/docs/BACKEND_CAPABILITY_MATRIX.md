# Backend Capability Matrix

枚举两个图谱后端（Zep Cloud / 本地 JSON store）的能力差异。本表是用户在
`.env` 里把 `ZEP_API_KEY` 设成占位符（即"全本地模式"）时**会实际看到的功能差**。

新增 `LocalGraphStore` 方法或新增 `GraphBuilderService` 方法时请同步本表。

| 能力 | Zep Cloud (`graph_id` 不带 `local_mirofish_` 前缀) | 本地后端 (`graph_id` 以 `local_mirofish_` 开头) | 决策 |
|---|---|---|---|
| 创建图谱 | ✅ `client.graph.create` | ✅ `LocalGraphStore.create_graph` | 等价 |
| 设置本体 | ✅ `client.graph.set_entity_types` | ✅ `LocalGraphStore.set_ontology` | 等价 |
| 添加文本（语义抽取） | ✅ Zep 自动抽取 | ⚠ LLM 兜底抽取（依赖 `LLMClient`） | Zep 无 LLM 成本，本地有 |
| 等待 episode 处理 | ✅ `_wait_for_episodes` 轮询 | ✅ no-op（本地写完即处理完） | 等价 |
| 获取图谱数据 | ✅ `fetch_all_nodes` + `fetch_all_edges` 分页 | ✅ 一次性读 JSON | 等价 |
| 获取单节点详情 | ✅ `client.graph.node.get` | ✅ `LocalGraphStore.find_node_anywhere` | 等价（带进程内缓存） |
| 获取节点的所有边 | ✅ `client.graph.edge.get_by_node` | ✅ 遍历本地 edges | 等价 |
| 删除图谱 | ✅ `client.graph.delete` | ✅ 删 JSON 文件 | 等价 |
| 图搜索 | ✅ `client.graph.search` | ✅ `LocalGraphStore.search` 全文匹配 | ⚠ 本地无语义检索，仅字面 |
| **市场人口层注入** (`upsert_market_population`) | ❌ **不支持** | ✅ 把合成 agent 注入图谱供前端可视化 | **only-local** |
| **图谱记忆动态更新** (`graph_memory_update`) | ✅ `ZepGraphMemoryUpdater` | ❌ **不支持**，被 `should_disable_graph_memory_update` 自动关闭 | **only-zep** |

---

## only-local 能力

### `LocalGraphStore.upsert_market_population`

`MarketPopulationGenerator` 生成的合成市场 agent 需要写入图谱，仅用于前端
可视化。Zep Cloud 没有等价能力，**也不打算补**——理由：

1. Zep 的图谱定位是"真实实体的 episodic memory"，把合成 agent 灌进去会污染未
   来的语义检索
2. 市场人口层主要作用是 OASIS 仿真侧的 profile 列表，图谱里的可视化是次要
3. 用户切到 Zep 的同时通常也接入了真实的市场参与者数据，不需要合成

`simulation_manager._sync_market_population_to_local_graph` 通过
`LocalGraphStore.is_local_graph_id(graph_id)` 守门，Zep 模式下直接 no-op，
无错。

## only-zep 能力

### 图谱记忆动态更新

`ZepGraphMemoryUpdater` 把仿真过程中 agent 的发言 / 互动作为新的 episode
写回 Zep，让长记忆持续生长。本地 store 没有"语义抽取"概念，写入只能落成
原始记录，意义不大。

API 层 `should_disable_graph_memory_update(graph_id)` 在以下两种情况下返
回 `True`：

1. `graph_id` 以 `local_mirofish_` 开头（本地后端）
2. `Config.ZEP_API_KEY` 是占位符（即便 graph_id 是真实 Zep id 也不可写入）

`/api/simulation/start` 命中此条件时：

- 把 `enable_graph_memory_update` 强制改回 `False`
- 在响应里附 `graph_memory_update_warning`（来自 `t('api.graphMemoryDisabledLocalBackend')`）
- 前端 `Step3Simulation.vue` 把 warning 渲染到日志区给用户看

---

## 检查清单（添加新能力时）

- [ ] 这个能力在另一边后端有等价实现吗？
- [ ] 如果只在一边，调用入口（API / 主流程）是否有 backend gate？
- [ ] backend gate 是否在响应里告知用户实际生效状态？
- [ ] 本表更新了吗？
