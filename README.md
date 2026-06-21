# Smart-Money Mirror — Wallet-Following Trading Agent

An on-chain copy-trading agent for BNB chain. It tracks a curated set of wallets
with a *proven* track record of realized PnL on volatile tokens, detects their
entries on-chain in near-real-time, filters them, and mirrors the high-conviction
ones — with its own risk engine bolted on top.

![Track](https://img.shields.io/badge/BNB_Hack-Track_1-F0B90B)
![Chain](https://img.shields.io/badge/Chain-BSC-yellow)
![Execution](https://img.shields.io/badge/Execution-PancakeSwap_V2%2FV3_direct-blue)
![Style](https://img.shields.io/badge/Style-smart--money_mirror-FF7043)
![Signals](https://img.shields.io/badge/Signals-on--chain_wallet_flow-7E57C2)
![MEV](https://img.shields.io/badge/tx-private_relay-red)

## What it is

A trading agent split into two decoupled halves that talk over a simple
file/JSON contract:

- **Brain (signal layer)** — maintains a vetted leaderboard of on-chain wallets
  with proven realized PnL on volatile tokens, watches their swaps on-chain, and
  turns a qualifying entry into a candidate order. Conviction comes from *who* is
  buying, not from a chart.
- **Executor (this skeleton, implemented)** — takes the brain's orders and runs
  them on BSC: market / limit buy+sell and conditional exits (take-profit,
  stop-loss, trailing). It does not think — it executes deterministically,
  idempotently, and survives restarts.

This repository currently ships the **execution engine** in full, plus the
contract the brain plugs into. The wallet-tracking signal layer is being wired in
next (see Status).

## The edge

Don't predict the move — follow the people who are already right.

1. **Proven wallets only** — the universe is a leaderboard built from realized
   PnL on volatile tokens, not followers or hype. Wallets that stop performing
   roll off.
2. **On-chain, near-real-time** — entries are detected from their swaps on-chain,
   so the agent can mirror inside the same window rather than after the candle.
3. **Filter, don't blindly copy** — confluence (multiple tracked wallets into the
   same token), position sizing, and an independent liquidity/safety check gate
   the copy; the risk engine then manages the exit on our terms.

The alpha is selection plus speed: a clean wallet set, fast detection, and a hard
downside cap.

## How it works

```mermaid
flowchart LR
    subgraph Brain
      L[wallet leaderboard<br/>proven realized PnL] --> W[watch swaps on-chain]
      W --> F{filters:<br/>confluence / safety / size}
      F -->|pass| O[order JSON]
    end
    O --> I[(orders_inbox)]
    subgraph Executor
      I --> D[daemon: triggers + gates]
      D --> E[PancakeSwap V2/V3<br/>direct swap, private relay]
      D --> ST[(SQLite state)]
    end
    ST --> Q[read-API: positions / fills]
    Q -.-> Brain
```

The two halves are decoupled on purpose: the brain writes order JSON into an
inbox atomically, the executor picks it up, fills it, and exposes a read-only
status API the brain polls for positions and fills. Either half can be iterated
or restarted without breaking the other.

## Status

**Implemented and battle-tested with real funds:**

- Direct PancakeSwap **V2 + V3** path (quote + swap, best-output routing).
- Trigger engine: limit entry + take-profit / stop-loss / trailing, in one place.
- Resident daemon: inbox orders + engine tick + risk gates, idempotent, with
  cancel, a per-wallet nonce manager + gap watchdog, RPC rotation, and SQLite
  persistence that survives restarts.
- Read-only status API for the brain; Telegram alerts + heartbeat.
- Unit-tested; many rounds of adversarial review behind it.

**Roadmap (the strategy layer — being wired in now):**

- Wallet leaderboard builder (realized-PnL scoring on volatile tokens) and a
  near-real-time on-chain swap watcher.
- Copy filters: multi-wallet confluence, token safety check, position sizing.
- Backtest harness replaying historical wallet flow.

The execution skeleton below is real and runnable today; the brain plugs into
the documented order/status contract.

## Architecture

| Module | Role |
| --- | --- |
| `src/models.py` | Order (market/limit, TP/SL/trailing) and result types |
| `src/config.py` | config, per-token slippage, whitelist, risk gates, secret loading |
| `src/rpc.py` | RPC pool: rotation + retry; primary on a private/MEV relay for tx |
| `src/nonce.py` | per-wallet nonce under lock, no collisions |
| `src/direct_adapter.py` | PancakeSwap V2/V3 swap: receipt check, fill from balance delta, multi-hop via USDT, gas reserve, preflight |
| `src/executor.py` | market-swap routing (direct / aggregator) |
| `src/triggers.py` | single engine: limit entry + TP + SL + trailing |
| `src/store.py` | SQLite (WAL): triggers, fill journal, idempotency |
| `src/daemon.py` | resident process: inbox orders + engine tick + risk gates |
| `src/runner.py` | one-shot run (+ `--dry-run`) |
| `src/status.py` | read-only state API for the brain |
| `src/register.py` | contest registration |
| `src/keystore.py` | scrypt keystore for private keys |
| `src/openocean.py` | optional aggregator routing |

## Usage

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
cp config.example.yaml config.yaml   # fill wallets/keys and number placeholders
```

Safe first run (spends nothing):

```bash
echo '{"wallet":"body1","side":"buy","token":"0x55d3...955","amount":0.01}' \
  | python -m src.runner --config config.yaml --order - --dry-run
```

Live (resident daemon):

```bash
python -m src.daemon --config config.yaml   # holds triggers + accepts orders
# brain drops order JSON into orders_inbox/*.json, results land in results/
```

Order schema (prices are BNB per 1 whole token):

```text
market buy           {"wallet":"body1","side":"buy","token":"0x..","amount":0.05}
+ TP/SL/trailing     add "take_profit_pct":0.2,"stop_loss_pct":0.1,"trailing_pct":0.08
limit buy            {...,"type":"limit","trigger_price":<BNB per token>}
sell all             {"wallet":"body1","side":"sell","token":"0x..","sell_all":true}
cancel               {"cancel":"<client_order_id>"}
```

Read state:

```bash
python -m src.status --config config.yaml [--wallet body1] [--fills 50]
```

Tests:

```bash
python -m pytest tests/ -q
```

## Risk and safety

- **MEV:** transactions are sent only through a private/MEV relay, so they do not
  hit the public mempool — sandwich protection at send time.
- **Risk gates:** max-drawdown and minimum-trades-per-day gates live in config
  (off by default until the strategy numbers are set).
- **Idempotency:** every order carries a `client_order_id`; replays and cancels
  are deduplicated, and state persists across restarts.

## The contest, briefly

Built for BNB Hack Track 1: an autonomous agent that registers on the contest
contract and trades on-chain within a hard max-drawdown constraint. This entry
borrows conviction from proven on-chain operators and caps its own downside with
the risk engine.

## Disclaimer

Experimental software for a hackathon. Trades real funds at your own risk. No
warranty; nothing here is financial advice. Keep private keys out of the repo —
this project loads them from env or an encrypted keystore, never from tracked
files.
