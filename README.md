# signal-backtest-harness

A **measurement-first** Solana memecoin signal-trading backtest harness (Python package
`memebot`). It evaluates a Telegram signal channel, but its first and most important job is
to **prove or disprove the channel's edge for ~$0 before any real capital is at risk.**

Why measurement-first: prior research shows
that following a *public* signal channel as a *downstream subscriber* is structurally
negative-EV — you fill after insiders/snipers/MEV and become their exit liquidity. The one
thing that can flip it positive is the **holding horizon**: you can't win sub-second snipe
races, but if the channel's edge lives on a slower horizon, latency stops mattering. We
don't guess that — the harness here measures it.

## Status: Phase 0 / 0.5 complete

A working, tested, honest backtest harness that takes a corpus of channel calls and prints
**net PnL by holding horizon** with the full dead-token denominator and bootstrap CIs.

## Quick start

```bash
uv sync --extra dev                      # Python 3.12+ venv + deps
uv run pytest                            # 20 tests

# End-to-end demo on LIVE data (no Telegram needed; uses real tokens):
uv run python scripts/make_sample_corpus.py --out runs/sample_corpus.json
uv run python scripts/backtest_channel.py --corpus runs/sample_corpus.json \
    --horizons 5m,15m,30m,1h --trade-size 2.0
```

The fill model is deliberately unflattering: it fills at the price `latency_seconds`
**after** the call, applies microcap slippage + venue fee + MEV drag + **fixed gas on both
the buy and the sell**, and scores any token with no exit liquidity as a **total loss**.
Tune everything in `config.toml`. Judge the result by **net profit factor** and a positive
CI lower bound — never win rate.

## Run on your real channel (Phase 0.5)

1. Connect the `chigwell/telegram-mcp` server (Telegram `api_id`/`api_hash`; the account
   must have joined the channel).
2. Pull the channel's message history and convert to a corpus:
   `signals_from_messages(messages, channel)` → `save_corpus_json(...)`
   (`src/memebot/ingest/telegram_mcp.py`).
3. `scripts/backtest_channel.py --corpus <that>.json` → read the table against the
   Go/No-Go gates in the plan.

## Layout

```
src/memebot/
  models.py          # shared dataclass contract (Signal, PriceSeries, FillResult, ...)
  config.py          # config.toml + .env loader
  parser/            # signal_parser.py — free-text call -> Signal (mint/ticker/side)
  data/              # geckoterminal.py (OHLCV), dexscreener.py (pairs/age)
  safety/            # rugcheck.py — mint/freeze authority, LP lock, holders, score
  sim/               # fill_simulator.py — the latency-honest fill + cost model
  backtest/          # horizon_backtest.py — PnL-by-horizon aggregator + report
  ingest/            # telegram_mcp.py — MCP/Telethon message dict -> Signal
scripts/             # backtest_channel.py (CLI), make_sample_corpus.py (live demo)
tests/               # deterministic unit tests
```

Engineering lessons captured during the build: `tasks/lessons.md`.
Pre-registration template for the on-chain extension: `tasks/prereg_onchain.md`.
