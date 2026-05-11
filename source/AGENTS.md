# AGENTS.md

## Project Shape

- Backend: `backend/`, Flask app started by `backend/run.py`.
- Frontend: `frontend/`, Vue/Vite app.
- Local graph and simulation artifacts live under `backend/uploads/`.
- Do not assume Zep is configured locally; placeholder `ZEP_API_KEY` should be treated as missing.

## Local Development

- Default frontend URL: `http://localhost:3001`.
- Default backend API URL: `http://localhost:5002`.
- `npm run dev` starts both services.
- Frontend API default is `http://localhost:5002`; override with `VITE_API_BASE_URL` if needed.
- Keep `frontend/vite.config.js` aligned with these defaults: dev server port `3001`, `/api` proxy target `http://localhost:5002`.
- Backend port is controlled by `FLASK_PORT`, defaulting to `5002`.

## LLM Providers

- `LLM_PROVIDER=openai` routes through the OpenAI-compatible HTTP client using `LLM_API_KEY`, `LLM_BASE_URL`, and `LLM_MODEL_NAME`.
- `LLM_PROVIDER=codex` routes shared LLM calls through `CodexCLIClient` and `codex exec`.
- `ONTOLOGY_LLM_PROVIDER` can override only ontology extraction; set it to `codex` for large documents or small local model context windows.
- `CODEX_MODEL` defaults to `gpt-5.5`; `CODEX_TIMEOUT_SECONDS` defaults to `300`.

## Verification

- Backend targeted/full tests: `cd backend; $env:PYTHONPATH='.'; uv run pytest`
- Frontend build: `cd frontend; npm run build`
- Quick Codex route smoke test:
  `cd backend; $env:PYTHONPATH='.'; @'
from app.utils.llm_client import LLMClient
print(LLMClient().chat_json([{"role": "user", "content": "Return JSON exactly: {\"ok\": true}"}]))
'@ | uv run python -`

## UI Notes

- 2026-05-07: `frontend/src/components/GraphPanel.vue` free-board graph should show node identity only. Do not render relation/edge labels such as `QUESTIONS_COMPLIANCE_RISK`, `MONITORS_MARKET_SIGNAL`, `REACHES_RETAIL_AUDIENCE`, or reintroduce the `Show Edge Labels` toggle unless the product requirement changes explicitly.

## Current Debugging Context

- A poor simulation result was traced to local Qwen context overflow, fallback graph extraction, empty `initial_posts`, and sparse OASIS actions.
- For this local workflow, regenerate graph, simulation, and report after switching to Codex; old artifacts remain sample-starved.
