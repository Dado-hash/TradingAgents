# TradingAgents

Multi-agent LLM financial trading framework. v0.2.5. Academic research project by Tauric Research.

## Architecture

5-stage LangGraph pipeline: **Analyst Team → Research Team → Trader → Risk Management → Portfolio Manager**

- `tradingagents/graph/trading_graph.py:TradingAgentsGraph` — main orchestration class
- `tradingagents/graph/setup.py` — graph construction / edge wiring
- Agents live under `tradingagents/agents/` per role

## Commands

```bash
pip install -e ".[dev]"          # install with test/lint deps
pip install -e ".[scheduled]"    # + yaml, fpdf2 (headless runner)
pip install -e ".[bedrock]"      # + AWS Bedrock support

ruff check .                     # lint (strict: E/W/F/I/B/UP/C4/SIM, E501 ignored)
ruff format .                    # DEFERRED — not adopted repo-wide yet; matches line-length=100

pytest -q                        # quiet mode (CI style)
pytest -m unit                   # fast isolated tests
pytest -m integration            # tests needing external services
pytest -m smoke                  # quick sanity checks

tradingagents                    # launch CLI (Typer)
python scripts/run_daily.py      # headless scheduled runner

python -c "import tradingagents, cli.main"  # smoke-test that runtime deps resolve
```

## Key quirks

- **`TRADINGAGENTS_*` env vars** override any key in `DEFAULT_CONFIG` via `_ENV_OVERRIDES` dict. Type-coerced from default's type. Empty vars silently ignored.
- **`.env` auto-loaded** at `import tradingagents` (`tradingagents/__init__.py` calls `dotenv.load_dotenv(usecwd=True)` with `override=False`). CLI prompts missing API keys and persists them to `.env`.
- **`tradingagents/__init__.py`** — runs on every import; respects pre-existing env vars.
- **Symbol normalization** (`dataflows/symbol_utils.py`): XAUUSD→GC=F, EURUSD→EURUSD=X, BTCUSD→BTC-USD, strips `+` suffix. `safe_ticker_component` prevents path traversal in filesystem paths.
- **Data vendor routing** (`dataflows/interface.py`): requests go to first configured vendor; no silent fallback to unconfigured vendors.
- **Decision memory** at `~/.tradingagents/memory/trading_memory.md` — reflection is deferred (fetches realized returns on next same-ticker run).
- **Report layout**: `reports/<TICKER>_<TIMESTAMP>/` with subdirs `1_analysts/`, `2_research/`, `3_trading/`, `4_risk/`, `5_portfolio/decision.md`, `complete_report.md`.
- **Lazy LLM imports** — `llm_clients/factory.py` imports provider modules lazily so just importing the factory doesn't pull in heavy SDKs.
- **`uv.lock` committed** — project uses `uv` but `uv run` is not required; `pip install -e .` works.

## Testing

- **50 test files** in `tests/`. Conftest auto-sets placeholder API keys (`_dummy_api_keys`) and deep-copies config before each test (`_isolate_config`).
- **No typecheck** configured (no mypy/pyright/pyre).
- No coverage config.

## CI (`.github/workflows/ci.yml`)

3 jobs triggered on push to `main` and all PRs:
| Job | Commands |
|-----|----------|
| test (py3.10–3.13, fail-fast:false) | `pip install -e ".[dev]"` → `pytest -q` |
| smoke-install (py3.12) | `pip install .` → `python -c "import tradingagents, cli.main"` |
| lint (py3.12) | `pip install "ruff>=0.15"` → `ruff check .` |

Concurrency: `ci-${{ github.ref }}` with cancel-in-progress.

## Docker

```bash
docker compose run --rm tradingagents
docker compose --profile ollama run --rm tradingagents-ollama
```

Multi-stage build (`python:3.12-slim`), non-root `appuser`, mounts `~/.tradingagents` volume.
