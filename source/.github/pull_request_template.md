<!--
  本仓库 PR 自检清单。请在合并前对每条改动逐项打钩或在该项后注明 N/A 与原因。
  详细约定见 docs/REVIEW_CONVENTIONS.md。
-->

## 改动概要

<!-- 用 1-3 句话说清楚为什么改、改了什么、影响半径 -->

## 自检清单

### 资源与并发
- [ ] 没有引入新的类级 `Dict` 状态；如果有，已用 `threading.Lock` / `RLock` 保护
- [ ] 所有打开的文件 / DB 连接 / 子进程在所有错误路径都被关闭（用 `with` / `closing` / `try-finally`）
- [ ] 同一个 `graph_id` 上的 read-modify-write 都在 `LocalGraphStore._lock_for(graph_id)` 临界区内
- [ ] 写 JSON / 配置文件用了"临时文件 + `os.replace`"原子写
- [ ] 后台 `threading.Thread` 里调用 `t()` / `get_locale()` 之前先注入了 `set_locale(captured_locale)`

### 外部依赖
- [ ] LLM / Zep 调用走了 `retry_with_backoff`、显式设置了 timeout 和 `max_tokens`
- [ ] 长时间运行的进程里发起 LLM 请求时已有节流（按 round / 按 agent / 全局上限）
- [ ] 走 `LLMClient()` 而不是直连 OpenAI SDK，以便用户切到 Codex 时也能透明工作

### 前后端契约
- [ ] 新增的 API 字段在 `frontend/src/api/*.js` 同步、相关 i18n key 在 `locales/zh.json` & `locales/en.json` 都补齐
- [ ] 影响 `Project / Task / Simulation / Runner / Report` 状态机的改动有测试覆盖新转移
- [ ] 引入的轮询 / 定时器 / 监听器在 `onUnmounted` 已清理

### 用户体验与契约透明
- [ ] 没有静默降级用户已勾选的功能；如果有，API 响应里带了 `*_disabled_reason` / `*_warning` 字段且前端有提示
- [ ] 没有改写用户输入；如果有（例如 prompt 前缀注入），它是 ① 幂等的、② 在响应中可观测、③ 在 docs 里写了
- [ ] 硬编码的领域知识（语言、行业、地区）都加了 `# DOMAIN-ASSUMPTION:` 注释标记

### 测试
- [ ] 修复了 bug 的改动同时加了红→绿回归用例
- [ ] 新增的 LLM / Zep 调用在测试里走 `FakeLLMClient` / 本地后端，不连真实网络
- [ ] 涉及子进程 / 文件 IO 的测试用 `tmp_path` 隔离，不污染 `backend/uploads/`

### 上下文成本（仅当涉及 LLM 调用）
- [ ] 单条工具/检索结果的入栈长度被截断（参考 `MAX_OBSERVATION_RESULT_CHARS`）
- [ ] 同名同参的工具调用会命中缓存而不是重打外部接口
- [ ] 给 LLM 的对话历史不会无界增长

## 测试

<!-- 列出你跑过哪些命令、覆盖了哪些场景；CI 通过的话直接 link to run -->
