# T BOT — multi-surface algorithmic trading system

> Production trading bot covering Kalshi prediction markets, OANDA forex, and IBKR equities. AI-driven prediction pipeline, signal-feed architecture for verbatim copy-trading, withdrawal-triggered commission model.

🌐 **Live demo:** [tbot.trade/demo](https://tbot.trade/demo)
📦 **Subscriber client:** [github.com/yourhandle/tbot-client](#) *(open source — runs locally on subscriber machines)*
🔒 **Operator-side bot:** private (strategy IP)

---

## What this is

T BOT is a four-surface automated trading system I built and run on a DigitalOcean droplet. It scans live markets, runs a multi-model AI prediction pipeline (Claude Sonnet + GPT-4o + Gemini ensemble for non-Kalshi surfaces), routes high-edge signals to brokers, and tracks calibration against realized outcomes.

In private beta as a signal subscription service for friends — they self-fund their own broker accounts, my client runs locally on their machine, I never touch their funds. Commission is 15% on realized profits with withdrawal-triggered crystallization and a high-water mark.

## The system at a glance

| Surface | Broker | Strategy | Status |
|---|---|---|---|
| **Kalshi ST** | Kalshi | Short-term prediction markets (weather, crypto, sports) | Live |
| **Kalshi LT** | Kalshi | Long-term political and macro events | Live |
| **OANDA FX** | OANDA | 8 major currency pairs with ATR brackets | Live (cutover in progress) |
| **IBKR STK** | Interactive Brokers | US equities via ibeam gateway | Live (funding in progress) |

**Operational footprint:** 6 PM2-managed processes on one Ubuntu droplet, FastAPI dashboard with subscriber feed API, append-only JSONL audit logs.

## What's interesting about it

### 1. Surface-isolated risk

Each surface has its own env file, bankroll, position cap, daily-loss circuit breaker, and concentration cap. A losing streak on FX cannot drain the STK allocation. This isolation is the foundation that makes the rest of the architecture safe to iterate on.

### 2. 10-gate risk chain

Every candidate signal traverses STOP-file check → dedup → per-market cap → direction filter → category suspension → risk validation → slippage check → minimum edge → minimum confidence → API cost cap. Most candidates are skipped — by design. The system is tuned for selectivity, not volume.

### 3. Multi-model AI ensemble (where applicable)

Kalshi LT uses a weighted ensemble of Claude Sonnet 4.6 (primary), GPT-4o, Gemini 2.0, and DeepSeek for probability estimation. Disagreement between models is itself a signal — high model variance flags markets where the prediction is unreliable.

### 4. Subscriber feed architecture

Signals are published to an append-only JSONL log served via `/api/signals/feed` (cursor-paginated, long-poll, 600s TTL). Subscribers run a Python client locally — they receive signals, apply their local size config, place orders on their own broker accounts, and POST `ack`/`executed`/`resolved` callbacks back to the operator. Operator sees aggregate performance only — no per-subscriber attribution.

### 5. Forward-only audit clarity

Every resolved trade record carries a `resolution_source` tag (`kalshi_resolution`, `nws_observation`, `oanda_candles`, `ibkr_position`, `sim`) so audits can partition real vs synthetic data cleanly. Audit tools accept `--since YYYY-MM-DD` so post-cutover metrics aren't polluted by historical dry-run noise.

### 6. ETA calibration

The system records `eta_hours` at execute time (distance-to-SL / atr_per_hour) and compares against actual `resolved_at - timestamp` per surface. Surfaces a late-resolution bias when present — currently ST shows 42% late-resolution rate, informing whether daily-loss circuit-breakers are calibrated for the actual turnover speed.

## Architecture

```
                          DigitalOcean droplet (Ubuntu)
   ┌────────────────────────────────────────────────────────────┐
   │  PM2 process supervisor                                    │
   │                                                            │
   │  st-bot-loop      ─→  Kalshi ST  (scan→predict→execute)   │
   │  st-lt-loop       ─→  Kalshi LT  (scan→predict→execute)   │
   │  st-forex-loop    ─→  OANDA FX   (scan→predict→execute)   │
   │  st-stocks-loop   ─→  IBKR STK   (scan→predict→execute)   │
   │  st-forex-resolver ─→  Closes FX positions when SL/TP hit │
   │  st-dashboard     ─→  FastAPI · subscriber feed · UI      │
   │                                                            │
   │  Surface env isolation:  st.env  lt.env  fx.env  stk.env   │
   └────────────────────────────────────────────────────────────┘
                            │
                            │  publishes to
                            ▼
   ┌────────────────────────────────────────────────────────────┐
   │  /api/signals/feed     (long-poll, cursor-paginated)       │
   │  /api/signals/{id}/ack         ←─┐                         │
   │  /api/signals/{id}/executed    ←─┤  subscriber callbacks   │
   │  /api/signals/{id}/resolved    ←─┘                         │
   └────────────────────────────────────────────────────────────┘
                            │
                            ▼
                   N subscriber clients
              (running locally — operator never sees broker creds)
```

See the [live deep-dive on tbot.trade/demo](https://tbot.trade/demo) for full ASCII diagrams of the signal pipeline (scan → research → predict → decide → execute → publish → resolve) and the subscriber feed flow.

## Screenshots

*Operator dashboard — per-surface allocations, P&L tracking, audit drawer, settings, subscriber aggregate panel.*

[screenshot-1.png]

*Live signal feed (operator side) — 4 surfaces, sub-strategy filters, edge/confidence per signal, dry-run vs live markers.*

[screenshot-2.png]

*Subscriber aggregate panel — operator view of aggregate performance across all subscribers, per-surface breakdown, no per-name attribution.*

[screenshot-3.png]

## Tech stack

- **Language:** Python 3.13
- **Runtime:** Ubuntu 22.04, PM2 process supervisor
- **Web:** FastAPI + Uvicorn
- **Data:** Append-only JSONL logs (no DB — single-writer pattern with fcntl advisory locks)
- **AI:** Anthropic Claude Sonnet 4.6 (primary), OpenAI GPT-4o, Google Gemini, DeepSeek
- **Brokers:** Kalshi API, OANDA v20 API, Interactive Brokers via ibeam HTTP gateway
- **External data:** NOAA NWS API (weather), Kalshi events/markets API
- **Edge:** Cloudflare (DNS, edge caching, email routing)
- **Frontend:** Vanilla HTML/CSS/JS — no framework

## Results to date

- 4 surfaces operational, multi-surface compound bankroll
- Subscriber feed architecture: end-to-end verified on real subscriber client (polling → routing → callbacks → operator aggregation)
- Demo page: edge-cached on Cloudflare globally
- Calibration tooling: per-surface audit scripts with `--since` scoping, ETA calibration across all four surfaces
- Risk: zero subscriber-fund custody by design — operator never holds broker creds

*Detailed performance metrics shared only with active beta subscribers via aggregate panel.*

## What I'd build next

1. **Native subscriber app** (Tauri desktop + Capacitor mobile) — current client is Python CLI; works but won't scale past technical users
2. **Strategy replay view** — "if you'd subscribed 30 days ago you'd have X" for prospective subscribers
3. **More AI models in the ensemble** — currently Claude/GPT/Gemini/DeepSeek; adding more provides better disagreement signal
4. **Surface expansion** — Polymarket (similar shape to Kalshi), commodities CFDs

## License & status

This is a private operational system. Code is not public. The [subscriber client](#) (the piece that runs on subscriber machines) is open source and lives in a separate repo — see linked above.

If you're hiring and want to talk about the design decisions, I'm available — contact via [your-email].
