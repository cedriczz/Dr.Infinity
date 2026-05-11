# ADR-001: Feature Gating Conventions

- **状态**：accepted
- **背景**：当前主线已有 1 个静默降级路径（`should_disable_graph_memory_update`），
  随着双图谱后端 / 双 LLM 提供商等差异化能力增加，未来还会出现更多"用户勾选了
  但后端不能执行"的情况。需要先把约定写下来，避免散点演化。

## 决策

### 1. 不允许静默降级

凡是用户在请求里显式启用、但后端因环境/能力原因决定不执行的功能，
**API 响应必须显式回传"实际生效状态 + 关闭原因"两个字段**。

字段命名采用统一前缀模式：

- 实际状态：`{feature}_enabled` (boolean)
- 关闭原因：`{feature}_warning` (string，i18n 后的人类可读文案)

例：

```json
{
  "graph_memory_update_enabled": false,
  "graph_memory_update_warning": "本地模式暂不支持..."
}
```

### 2. warning 文案必须走 i18n

不允许在 Python 代码里硬编码中文/英文，统一通过 `t('api.xxxDisabled...')`
检索。模板放 `locales/{zh,en}.json` 的 `api.*` 命名空间。

### 3. 前端必须把 warning 渲染出来

任何前端组件调用了带 gating 行为的 API 后，必须把 `*_warning` 字段以可见
方式呈现。当前的最小标准是"加到日志区"，未来可以升级为弹窗或 toast。

### 4. gating 判定函数必须可独立测试

把判定逻辑抽成纯函数（如 `should_disable_graph_memory_update`），测试时
不需要启动 Flask 应用。每个 gating 至少有：

- 命中条件 1（如本地后端）→ 返回 True
- 命中条件 2（如占位 key）→ 返回 True
- 不命中（真实组合）→ 返回 False
- 边界（空输入）→ 行为定义且测试

参考 `tests/test_simulation_api_helpers.py`。

## 不在范围内

- 鉴权/授权层面的"用户没权限做这个" — 那是 403 问题，不是 gating
- 配额/限流 — 那是另一个模式，应当独立 ADR

## 已知应用此约定的位置

| 路径 | gating 函数 | warning 字段 | 前端展示 |
|---|---|---|---|
| `/api/simulation/start` `enable_graph_memory_update=true` | `should_disable_graph_memory_update` | `graph_memory_update_warning` | `Step3Simulation.vue` 日志区 |

新增 gating 时把这一行加进表里。
