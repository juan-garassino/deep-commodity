# deepCommodity

An agentic quant trader where Claude Code is the agent and Python is the guardrail layer: the LLM proposes trades, code-enforced gates decide whether they happen.

> **Research / paper trading only. Not financial advice.** No live trading has been enabled, and this repo contains no performance claims (see [Results](#results)).

## What and why

Most "LLM trader" demos let the model call a broker directly. Here there are no agent classes in Python: scheduled headless Claude sessions ("routines") read markdown state, call CLI tools in `tools/` that fetch data and place orders, and write results back to append-only logs. Every order passes through a single chokepoint, `deepCommodity/guardrails/preflight.py`, fed by an authoritative broker snapshot, so risk limits do not depend on the agent remembering them. Design rationale: [AGENTIC-QUANT.md](AGENTIC-QUANT.md); operating contract: [AGENT-INSTRUCTIONS.md](AGENT-INSTRUCTIONS.md).

## Highlights

- **Code-enforced risk gates.** Position (5% NAV), sector (30%), gross leverage (1.0x), cash floor (10%), daily and per-bucket caps, halt checks and the live-authorization gate live in `preflight`; `place_order.py` and `risk_check.py` are thin wrappers over it and fail closed. Tests: `tests/test_preflight.py`, `test_guardrails.py`, `test_place_order_gates.py`, `test_limits_enhanced.py`.
- **Three-bucket decisions** (anchor / theme / hidden gem) with a 3-new-positions-per-day cap. Themes need at least 2 distinct evidence source-types, defined in [TRADING-STRATEGY.md](TRADING-STRATEGY.md) and `deepCommodity/universe/themes.yaml`. The confidence and thesis-length floors are enforced by the routine prompts, not by code.
- **Multi-stream signal tools:** news, insider Form 4, SEC 8-K, earnings, on-chain, cross-asset correlation, and Fed-funds implied rates (`tools/fetch_*.py`, `correlation_matrix.py`). News text is sanitized before the agent reads it (`guardrails/sanitize.py`).
- **Several forecasters behind one router** (`tools/forecast.py`): rule-based, price/order-flow/fused transformers, news model, and a macro-contextual transformer designed to be leakage-safe (publication-lag-shifted macro features). Trained models are optional and need torch.
- **Layered halts:** `KILL_SWITCH` file, `DC_HALT` / `TRADING_MODE=halt` env vars, and a drawdown breaker (`tools/check_drawdown.py`).
- **Offline research:** walk-forward backtester reusing the live risk gates (`deepCommodity/backtest/`) and portfolio sleeves (cross-sectional L/S, funding carry) in `deepCommodity/portfolio/`.

## Quickstart

Requires Python 3.11+ (CI uses 3.12). The mocked test suite needs no API keys or broker accounts.

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pip install pytest-timeout
pytest -q tests/test_preflight.py tests/test_guardrails.py tests/test_place_order_gates.py
```

`make dc-test` runs the full suite (`pytest tests/ -q`). The transformer, contextual, portfolio-research and serving tests additionally need `torch` (CI installs the CPU wheel `torch==2.2.2`) and a compatible `httpx` for the FastAPI test client.

To run the agent loop you need your own accounts: Alpaca **paper** keys, Binance testnet keys, `OPENAI_API_KEY` for news, plus optional Finnhub/FRED/Telegram keys. Copy `.env.sample` to `.env` (gitignored) and see [CLAUDE.md](CLAUDE.md) for the variable list. Binance geo-blocks some cloud egress, so crypto is intended to run on a VM ([deploy/README.md](deploy/README.md)).

## Project structure

```
deepCommodity/
  guardrails/   preflight (chokepoint), limits, kill switch, sanitize, circuit breaker
  execution/    Alpaca / Binance adapters, authoritative portfolio snapshot
  universe/     themes.yaml + loader
  model/        LSTM, price / order-flow / fused / contextual transformers, news model
  backtest/     walk-forward engine reusing live risk gates
  portfolio/    research sleeves + risk portfolios (portfolios.yaml)
tools/          CLI tools the agent calls (JSON to stdout)
tests/          mocked test suite (MockBroker in _mocks.py)
serving/        FastAPI inference service + Dockerfile
deploy/         VM kit: cron, systemd, preflight, reactive watcher
notebooks/      Colab training entry point
docs/superpowers/specs/   design specs
*-LOG.md, WEEKLY-REVIEW.md   append-only agent journals
```

## Results

No backtest outputs or performance statistics are committed, so none are claimed. What the logs do show:

- [TRADE-LOG.md](TRADE-LOG.md): 5 entries, all from May 2026 gate checks and one paper smoke order (a 1-share AAPL buy on Alpaca paper). It is not a performance record.
- [WEEKLY-REVIEW.md](WEEKLY-REVIEW.md): the 2026-08-16 review reports 0 trades and 8 consecutive 0-trade passes, attributed to unreachable signal streams from cloud egress and a conservative forecaster. It proposes (does not apply) strategy edits.

## Tech stack

Python, ccxt, alpaca-py, pandas / scikit-learn / statsmodels, PyTorch (optional, for the transformers), FastAPI, pytest, Claude Code routines, systemd/cron.

## Status

- Gates and tools are code-complete and covered by mocked tests; paper trading has produced no meaningful track record yet.
- Live trading is gated: it requires `TRADING_MODE=live`, `DAILY_DECISION_AUTHORIZE_LIVE=true`, `--confirm-live`, and NAV under `DC_MAX_NAV_USD`.
- Transformer models await Colab training; the contextual model is intended to run in shadow until its walk-forward eval beats the rule-based baseline.
- Known gaps (from the weekly review): VM deployment for crypto signals, a news fallback provider, and forecaster tuning.
- Short/leveraged portfolio sleeves are offline research only.

## License

No LICENSE file is present in this repository.
