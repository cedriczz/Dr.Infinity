# Local Development

## Services

Run both services from the project root:

```bash
npm run dev
```

Default local URLs:

- Frontend: `http://localhost:3001`
- Backend API: `http://localhost:5002`

The backend reads the root `.env`. The frontend reads `VITE_API_BASE_URL` from the process environment when provided and otherwise defaults to `http://localhost:5002`.
The Vite dev server is pinned in `frontend/vite.config.js` to port `3001`, and its `/api` proxy target must stay aligned with the backend default port `5002`.

Uploads accept PDF, Markdown, TXT, and ZIP archives. ZIP files are unpacked on the backend, filtered to supported document types, and then fed through the same text extraction flow as normal uploads. `MAX_CONTENT_LENGTH` controls the request body limit in bytes and defaults to `524288000` (500MB).

## LLM Provider Modes

Use OpenAI-compatible HTTP mode for hosted APIs:

```env
LLM_PROVIDER=openai
LLM_API_KEY=your_api_key
LLM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_MODEL_NAME=qwen-plus
ONTOLOGY_LLM_PROVIDER=openai
```

Use Codex CLI mode for large local workflows where a local model context window is too small:

```env
LLM_PROVIDER=codex
ONTOLOGY_LLM_PROVIDER=codex
CODEX_MODEL=gpt-5.5
CODEX_TIMEOUT_SECONDS=300
```

Codex mode routes shared LLM calls through `backend/app/utils/codex_cli_client.py`. The CLI must be available on `PATH` as `codex`, `codex.cmd`, or `codex.exe`.

## Troubleshooting Sparse Simulations

If the final report is shallow, inspect the upstream artifacts before changing the report prompt:

- Graph extraction logs for context overflow or fallback graph creation.
- `backend/uploads/simulations/<simulation_id>/simulation_config.json` for `agent_configs` count and `event_config.initial_posts`.
- `twitter/actions.jsonl` and `reddit/actions.jsonl` for actual action counts.
- Profile generation logs for LLM context errors and rule-based fallback.

Old graph and simulation artifacts do not improve after changing LLM provider. Regenerate the graph, simulation config, simulation run, and report so the larger-context provider affects every stage.

## Verification

```bash
cd backend
$env:PYTHONPATH='.'
uv run pytest
```

```bash
cd frontend
npm run build
```
