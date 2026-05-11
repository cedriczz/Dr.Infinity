# ADR-002: LLM Provider Routing (OpenAI vs Codex CLI)

- **状态**：accepted
- **背景**：用户在生产中希望把所有"预测类"LLM 调用切到本地 Codex CLI（成本、隐私、长上下文）。
  此前 `Config.ONTOLOGY_LLM_PROVIDER` 只覆盖本体生成，其他调用点（`oasis_profile_generator`、
  `simulation_config_generator`、`report_agent`、`simulation_runner.fallback_interview`）都
  绕开开关、写死 `LLMClient`。

## 决策

### 1. 单一全局开关 `Config.LLM_PROVIDER`

环境变量 `LLM_PROVIDER` 取值 `openai`（默认）或 `codex`。模块级开关
（`ONTOLOGY_LLM_PROVIDER`）可以独立覆盖，不显式设置时跟随全局值。

### 2. 路由逻辑下沉到 `LLMClient`

`LLMClient.__init__` 读 `Config.LLM_PROVIDER`：

- `openai`：用 OpenAI Python SDK 走 `Config.LLM_API_KEY` + `Config.LLM_BASE_URL`
- `codex`：懒构造 `CodexCLIClient`，把 `chat()` / `chat_json()` 透明转发

调用方**不需要**知道当前是哪条路径。这意味着：

- 现有所有 `LLMClient()` 调用点自动跟随全局开关
- 想新加一个调用点 → 直接写 `LLMClient().chat(...)` 即可

### 3. `Config.validate()` 在 codex 模式下不再要求 `LLM_API_KEY`

只有 `LLM_PROVIDER == "openai"` 才校验 API key。Codex CLI 不需要 key。

### 4. Codex 客户端的限制

| LLMClient 方法 | OpenAI | Codex CLI |
|---|---|---|
| `chat()` | ✅ 全功能 | ✅ 但 `temperature/max_tokens/response_format` 不生效（CLI 自管推理预算） |
| `chat_json()` | ✅ 走 `response_format={"type":"json_object"}` | ✅ 通过 prompt 注入 + JSON 解析兜底 |
| 原生 `tool_calls` 字段 | ✅ | ❌ |

`ReportAgent` 的 ReACT 循环使用**文本协议**解析 `<tool_call>...</tool_call>`，
不依赖 OpenAI 的原生 tool_calls 字段，因此与 Codex 兼容。新加的调用点如果
依赖原生 tool calling，需要在文档里明确"只支持 OpenAI"。

### 5. 部署侧的注意事项

切到 codex 时：

- 确保 `codex` 命令在 PATH 中（`shutil.which`）
- 设置 `CODEX_MODEL`（默认 `gpt-5.5`）和 `CODEX_TIMEOUT_SECONDS`（默认 300）
- 大幅增大单次调用延迟（CLI 启动 + 推理）→ 评估前端轮询间隔是否足够

## 已知应用此约定的位置

所有走 `LLMClient()` 的调用点都已隐式应用，无需逐个改代码。明确受影响的关
键路径：

- `OntologyGenerator`
- `OasisProfileGenerator`
- `SimulationConfigGenerator`
- `MarketPopulationGenerator`（不直接调 LLM，仅用于 cohort 模板）
- `ReportAgent`（ReACT + 章节生成）
- `SimulationRunner._fallback_interview_response`（offline interview 兜底）
- `ZepToolsService.insight_forge`（生成子问题）

## 不在范围内

- 多 provider 路由（OpenAI + Anthropic + Bedrock）—— 当前只支持二选一
- per-call 切换（同一进程内某些调用走 OpenAI、另一些走 Codex）—— 应用层改 `LLM_PROVIDER` 即可
