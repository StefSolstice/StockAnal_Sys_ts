# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

智能分析系统 (Intelligent Analysis System) v2.2.0 — a Python/Flask web application for AI-driven Chinese A-share stock analysis. It integrates a LangGraph multi-agent pipeline alongside traditional technical/fundamental analysis engines. All AI calls use an OpenAI-compatible API (not necessarily OpenAI itself).

## Running the Application

```bash
# Direct (development)
python run.py                      # runs on http://localhost:8888

# Via management script
bash scripts/start.sh start        # background, PID tracked in data/.server.pid
bash scripts/start.sh stop
bash scripts/start.sh restart
bash scripts/start.sh status
bash scripts/start.sh monitor      # auto-restarts on crash
bash scripts/start.sh logs

# Docker (production)
docker-compose up                  # includes Redis sidecar
```

No test suite exists despite `pytest` being in requirements. There is no linter configuration.

## Required Setup

Copy `.env-example` to `.env` and fill in at minimum:

```
OPENAI_API_KEY=...
OPENAI_API_URL=https://.../v1    # any OpenAI-compatible endpoint
OPENAI_API_MODEL=...
NEWS_MODEL=...                    # model used for news/Q&A (supports web search)
```

Optional keys (system degrades gracefully without them):
- `TAVILY_API_KEY`, `SERP_API_KEY` — web search fallbacks (DuckDuckGo is tried first, free, no key needed)
- `FINNHUB_API_KEY` — US stock data
- `USE_REDIS_CACHE=true` + `REDIS_URL` — persistent cache (falls back to in-memory)
- `USE_DATABASE=true` + `DATABASE_URL` — SQLite/Postgres persistence (defaults off)
- `USE_AGENT_SYSTEM=true` — enables the LangGraph agent pipeline (default true)

## Architecture

### Core Infrastructure (`app/core/`) — All Singletons

Every core service uses the singleton pattern; obtain instances via `get_*()` factory functions, never construct directly.

| Module | Purpose |
|---|---|
| `ai_client.py` | Single OpenAI client with 180s timeout, 2 retries. All AI calls go through `chat_completion()` here. |
| `data_provider.py` | `DataProvider` — unified data layer over AKShare (primary) and BaoStock (fallback), with 200ms rate limiting and automatic cache-through. |
| `cache.py` | `UnifiedCache` — Redis-first, in-memory fallback. All cache keys are prefixed `stockanal:`. |
| `search.py` | `search_web()` — cascades DuckDuckGo → Tavily → SERP. |
| `agent_memory.py` | Persists analysis history to `data/agent_memory/<code>_history.json`, max 50 entries per stock. Supports TF-IDF semantic search. |
| `event_bus.py` | In-process pub/sub for agent events (`analysis.started`, `analysis.completed`, `risk.alert`, etc.). |
| `fallback_manager.py` | Wraps adapters in a chain; tries each until one succeeds. |

### Data Adapters (`app/adapters/`)

`BaseAdapter` defines the contract: `get_stock_history`, `get_index_stocks`, `get_stock_info`, `get_financial_data`, `health_check`. Both `AkshareAdapter` and `BaostockAdapter` implement it. `DataProvider` routes through `FallbackManager([akshare, baostock])`. Methods exclusive to AKShare (capital flow, industry, north-bound funds) bypass fallback and call `self.akshare` directly.

### Multi-Agent Pipeline (`app/agents/`)

Orchestrated by LangGraph. Entry point: `coordinator.run_agent_analysis()`.

**Shared state** is `StockAnalysisState` (TypedDict in `state.py`), passed through every node.

**Graph topology** is depth-controlled (1–5):
```
depth 1:  technical → decision
depth 2:  + fundamental → capital_flow
depth 3:  + sentiment
depth 4:  + bull → bear  (debate)
depth 5:  + risk → investors (Buffett/Munger/Lynch/Damodaran)
          always ends with: → decision → reflection → END
```

**Every agent** extends `BaseStockAgent` (abstract), which injects `self.client`, `self.model`, and `self.data_provider`. Agents implement a static `analyze(state) -> state` method (matching LangGraph node signature).

**Self-improvement loop**: After each analysis, `ReflectionAgent` writes a reflection to `data/agent_reflections/`. If ≥3 reflections exist for a stock, `StrategyEvolver` rewrites the strategy prompt in `data/agent_strategies/`. The next run injects this prompt as the initial system message.

**Human-in-the-Loop** (`hitl.py`): High-risk decisions block until manually approved via `POST /api/agent_submit_approval`. Low/medium risk auto-approve.

### Traditional Analysis Engines (`app/analysis/`)

Independent of the agent pipeline; used directly by Flask routes. `StockAnalyzer` is the central engine — it holds all technical indicator parameters (`ma_periods`, `rsi_period`, `bollinger_period`, etc.) and is configured in its `__init__`. Scoring is 100-point weighted: 40% technical, 40% fundamental, 20% capital flow.

### Web Layer (`app/web/`)

- `web_server.py` — Flask app, all routes, background task threads, Swagger at `/api/docs`
- `auth_middleware.py` — optional API key / HMAC auth (configured via `API_KEY` / `HMAC_SECRET` env vars)
- `industry_api_endpoints.py` — separate blueprint for industry endpoints
- Templates use Bootstrap 5 + ApexCharts, with `layout.html` as the base

### MCP Server (`app/mcp/`)

Exposes 5 stock data tools (`get_stock_history`, `get_technical_analysis`, `get_financial_data`, `get_capital_flow`, `search_news`) via `POST /api/mcp/call`. This allows external LLM tools to query the system.

## Key Conventions

**Adding a new agent**: subclass `BaseStockAgent`, implement `analyze(state: Dict) -> Dict`, add a node in `coordinator.build_analysis_graph()` at the appropriate depth level.

**Adding a new data method**: implement it on `BaseAdapter`, add to both `AkshareAdapter` and `BaostockAdapter`, then expose it on `DataProvider`. If only AKShare supports it, add it under the `# akshare专有方法` section and call `self.akshare` directly.

**All AI calls** must go through `app/core/ai_client.chat_completion()` — never instantiate `OpenAI` directly in feature code.

**Cache TTLs**: stock history = 1800s, stock info/financials = 3600s. Cache auto-clears at 16:30 daily (post-market close).

**File-level docstring format**: every module starts with `Input/Output/Pos` comments (see any `app/core/*.py`). When modifying a file, update its header comment and note in its folder's `README.md`.

**Stock code formats**: A-share = 6 digits, HK = 4-5 digits, US = 1-5 letters. Validated in `web_server.validate_stock_code()`.

**Persistent data directories** (created at runtime, gitignored):
- `data/agent_memory/` — per-stock analysis history
- `data/agent_reflections/` — reflection logs
- `data/agent_strategies/` — evolved strategy prompts
- `data/news/` — cached news by date
- `logs/` — application logs
