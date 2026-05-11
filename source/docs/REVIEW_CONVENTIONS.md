# Review Conventions

本文档是 PR checklist 背后的约定，把"为什么这么要求"写一次，避免每次 review 重复解释。

---

## 1. 并发与资源

### 1.1 类级共享状态必须加锁

`SimulationRunner._run_states / _processes / _action_queues / ...` 这类
"`Dict[simulation_id, X]`"模式在 Flask `threaded=True` 下被多个请求线程并发读
写。新增同类状态时遵循已经定型的两步：

```python
class MyService:
    _state: Dict[str, X] = {}
    _state_lock = threading.RLock()

    @classmethod
    def get(cls, key):
        with cls._state_lock:
            return cls._state.get(key)
```

### 1.2 文件 IO 用原子写 + 临时文件

落盘 JSON 状态文件时，参考 `SimulationRunner._save_run_state` 与
`LocalGraphStore._write_graph` 的"`tmp_path` + `os.replace`"模式。崩溃中途
不会留下损坏的 JSON。

### 1.3 同图谱上的 read-modify-write 必须串行化

`LocalGraphStore` 提供了 per-graph 的 `_lock_for(graph_id)` 锁。任何
`read → mutate → write` 三步组合（如 `set_ontology`、`add_text_batches`、
`upsert_market_population`）必须包在该锁里。**临界区内不要发 LLM、不要做
长 IO**——`add_text_batches` 已经把 LLM 抽取阶段拆出来在锁外做。

### 1.4 后台线程必须显式继承 locale

```python
captured = get_locale()
threading.Thread(target=worker, args=(captured,), daemon=True).start()

def worker(locale):
    set_locale(locale)
    ...
```

否则 `t()` 在线程里只能读到 fallback。

---

## 2. LLM 调用

### 2.1 始终通过 `LLMClient()`

不要直接使用 `openai.OpenAI(...)` 或 `subprocess.run(["codex", ...])`。
`LLMClient` 在初始化时根据 `Config.LLM_PROVIDER` 自动路由到 OpenAI 或
Codex CLI；你只需要写 `client.chat(...)` / `client.chat_json(...)`。

这也意味着：用户在生产里把 `LLM_PROVIDER` 改成 `codex`，所有走
`LLMClient()` 的调用点都会跟着切换，**无需改代码**。

### 2.2 永远设置 `max_tokens` 和超时

Codex CLI 有 `CODEX_TIMEOUT_SECONDS`；OpenAI 调用要么显式 `max_tokens`，
要么明白上游 SDK 默认值。报告生成里观察到的现象是"省一个参数烧 4× 成本"。

### 2.3 工具调用结果要缓存与截断

`ReportAgent` 已经在章节级使用 `section_tool_cache` 和
`MAX_OBSERVATION_RESULT_CHARS = 8000`，避免：

- LLM 重复同名同参调用 → 重复打 Zep + 重复占用上下文
- Zep 偶发返回大段 JSON → 让每轮上下文成倍膨胀

新加的 tool / 检索路径要遵循同样模式。

### 2.4 高 fan-out 节点查询要进程内缓存

`ZepToolsService.get_node_detail` 维护 `_node_detail_cache`，让
`insight_forge` 等 N+1 模式退化为 N 次首查 + O(1) 命中。**包括缓存 None**
（已确认不存在的节点也不要反复打底层）。

---

## 3. 用户契约透明

### 3.1 没有静默降级

如果后端因为环境原因（占位 key、本地后端等）禁用了用户已勾选的功能，
**API 响应必须显式回传**：

```python
response_data['graph_memory_update_enabled'] = enable_graph_memory_update
if graph_memory_update_warning:
    response_data['graph_memory_update_warning'] = graph_memory_update_warning
```

前端必须把 `*_warning` 渲染出来。`Step3Simulation.vue` 是当前的参考实现。

### 3.2 改写用户输入要可观测且幂等

`optimize_interview_prompt(prompt)` 给所有 interview prompt 注入中文前缀。
新加的"用户输入改写"必须做到：

1. **幂等**：再次输入已改写过的 prompt 不会重复改写（`startswith` 检查）
2. **可观测**：响应里把改写后的 prompt 露出来（决策 ADR 待写，目前未实现）
3. **文档化**：在这里加一段说明它在哪些路径生效

### 3.3 领域硬编码必须打标

`MarketPopulationGenerator` 内嵌了"韩国加密市场"cohort、 `LLMClient` 的中
文 system prompt、`offline interview` 的中文 fallback 文案——这些都属于领
域假设。新加的同类常量请加注释：

```python
# DOMAIN-ASSUMPTION: 韩国加密市场散户特征，
# 切换到其他市场（如美国股市、日本散户）需要替换此 cohort
KOREAN_RETAIL_COHORTS: ...
```

便于日后做 i18n 时 `grep -r 'DOMAIN-ASSUMPTION' .` 一把抓出来。

---

## 4. 测试与回归

### 4.1 修复 bug 必先写 red 用例

不补回归用例的 bug 修复 PR 应当被拒绝。参考
`tests/test_simulation_api_helpers.py` 的 ``test_should_disable_*`` 系列。

### 4.2 测试零外部依赖

- LLM 调用走 `tests/conftest.py` 里的 `FakeLLMClient`
- Zep 调用走 `LocalGraphStore` 后端（`monkeypatch.setattr(... GRAPH_DIR ...)`)
- 文件 IO 走 `tmp_path` fixture

### 4.3 并发用例打 `slow` marker

用 `@pytest.mark.slow` 标记，CI 主流程不跑，nightly 跑：

```python
@pytest.mark.slow
def test_simulation_runner_under_concurrent_starts(...):
    ...
```

---

## 5. 前端

### 5.1 d3 重新渲染前 `interrupt()`

```js
svg.selectAll('*').interrupt().remove()
```

否则 d3 会对已分离 DOM 派发 tick / end 事件，造成内存泄漏 + 视觉抖动。

### 5.2 路由切换前清理定时器

```js
onUnmounted(() => {
  if (pollTimer) clearInterval(pollTimer)
})
```

页面级 polling（`Process.vue → MainView.vue`、`SimulationRunView.vue`、
`ReportView.vue`、`InteractionView.vue`）都必须满足这条。

### 5.3 v-for 不要用 index 作 key

文件上传列表、agent 列表、章节列表等 reorderable 列表必须用稳定 id。
