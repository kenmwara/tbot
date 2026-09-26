# T BOT — autonomous trading on Kalshi prediction markets

> A live-money trading system that has to earn every lane it runs: a 14-layer risk guard chain, books reconciled to the exchange every night, and its own detection-and-response stack.

🌐 **Live demo:** [tbot.trade/demo](https://tbot.trade/demo)
🛡️ **Security operations:** [github.com/kenmwara/tbot-security](https://github.com/kenmwara/tbot-security)
📦 **Subscriber client:** [github.com/kenmwara/tbot-client](https://github.com/kenmwara/tbot-client) *(open source, the original signal-feed model)*
🔒 **Operator-side bot:** private (strategy IP)

---

## Screenshots

<p>
  <img src="https://tbot.trade/portfolio/img/tbot-dash.jpg?v=20260926" width="400" alt="T BOT surface roster: the live Kalshi weather surface, and LT, STK, CRYPTO, FX and the Claude lane retired on evidence with their reasons">
  &nbsp;&nbsp;
  <img src="https://tbot.trade/portfolio/img/tbot-st.jpg" width="400" alt="tbot.trade ST surface page: bankroll mark-to-market, banked P&L, max drawdown, equity curve, positions, performance and the Kalshi settlement resolver">
  &nbsp;&nbsp;
  <img src="https://tbot.trade/portfolio/img/tbot-soar.jpg" width="400" alt="ops.tbot.trade/soar in demo mode: the security console with verdict, posture tiles, incident timeline and event feed">
</p>

*Left: the operator dashboard, one card per surface. Middle: the live weather surface, bankroll, P&L, drawdown and the settlement resolver. Right: the security console in its synthetic demo mode.*

## What this is

T BOT trades Kalshi prediction markets on its own, around the clock, from one Ubuntu droplet. The live surface is weather: daily temperature contracts across US cities, settled against the National Weather Service's own observations. I built it and run it alone.

It started wider. Forex (OANDA), macro events (Kalshi), US equities (IBKR) and crypto (Kraken) were each built, run on real or paper money, and measured. **None of them showed an edge that survived fees, so all four were retired on the evidence.** They stay on the dashboard so the decision can be re-examined, not forgotten.

A copy-trade pilot is in private beta: a subscriber connects their own Kalshi account by API key (encrypted at rest, AES-GCM, revocable at any time), and the engine mirrors the operator's trades at proportional size. The subscriber keeps custody. Commission is 15% of realised profit, billed through Stripe. The financial-services app listing is held for regulatory counsel rather than shipped and argued about later.

## The surfaces

| Surface | Venue | What it trades | Status |
|---|---|---|---|
| **ST weather** | Kalshi | Daily temperature contracts | **Live** |
| LT macro | Kalshi | Longer-dated macro events | Retired 2026-09 (no edge) |
| STK | IBKR | US equities, long-only trend | Retired 2026-09 (trailed buy-and-hold) |
| CRYPTO | Kraken | Dip-buying | Retired 2026-09 (edge smaller than fees) |
| FX | OANDA | Currency pairs | Retired 2026-06 (flat after 232 real trades) |

## What's interesting about it

### 1. A lane pays for itself, or it is recalibrated

Every lane is billed nightly for its real exchange fees **plus the Claude spend that made its decisions**. When the bill exceeds the return, the lane is flagged for recalibration. The rule does not switch anything off by itself, because that is a risk decision a human makes. It is how a Claude-driven prediction lane was retired: measured against its own inference cost, it lost money. The model is only worth its price if the decisions it makes are.

### 2. A 14-layer guard chain

Every candidate order passes fourteen sequential checks before it exists: leg-count limits, event-calendar blackouts, expiry proximity, slippage since the scan, a fee-aware edge gate with an overconfidence cap, confidence, signal strength, then the kill switch, open-position and exposure caps, and available capital at the moment of the order. Every layer is an environment variable, reversible without a deploy, and most candidates are refused. The system is tuned for selectivity, not volume.

### 3. Every knob is replayed before it ships

A proposed change to a risk limit is first replayed against logged history and graded on real settlements, with the count, win rate and expected value reported. One proposed loosening would have opened 67 markets in a week at a net loss, so it was refused. A new edge starts at capped size with a scheduled verdict date. When a single bad weather reading once triggered a "certain" lock, the fix (two consecutive readings) was replayed first: it kept 59 calls with zero losses and refused 18, all of which would have won or were still pending.

### 4. The books come from the exchange

A nightly job pulls every fill, settlement, deposit and withdrawal from the exchange's own API, and the result has to close to the cash the exchange reports, within $5. Getting it to close taught three things the documentation had wrong: a NO purchase is reported as `action=sell, side=no`; there is no settlement fee (so two of the three in-repo fee estimators were off, one at half and one at 1.6×); and YES/NO pairs net into cash the moment they form. It also overturned an earlier documented lifetime figure. The exchange's number is the only one that counts now.

### 5. Security is its own system

Intrusion detection watches for unknown SSH keys, file changes, new ports, and, most usefully, **contracts filled on the exchange that the bot never logged**. That is how a misused API key would show, and it engages the kill switch automatically. Every event lands in a SIEM, a SOAR console holds the response playbooks, and a red team attacks the system from GitHub Actions every night. Full write-up: [tbot-security](https://github.com/kenmwara/tbot-security).

### 6. Rebuildable by design

In August 2026 the droplet was destroyed with no snapshot. It was rebuilt from git and the secrets vault to live trading in one evening. Backups are now encrypted nightly and the restore is proven every night rather than assumed.

## Architecture

```
                         DigitalOcean droplet (Ubuntu)
   ┌──────────────────────────────────────────────────────────────┐
   │  st-bot-loop    scan → research → decide → guard → execute   │
   │                 Kalshi weather, NWS observations              │
   │  st-dashboard   FastAPI · operator UI · subscriber API        │
   │  cron           settlement resolver · exchange books rebuild  │
   │                 · lane economics · intrusion detection        │
   │                 · encrypted backup · Morning Verify           │
   └──────────────────────────────────────────────────────────────┘
          │ events (signed)                    ▲ playbooks
          ▼                                    │
   ┌──────────────────────────────────────────────────────────────┐
   │  Cloudflare: ingest worker → D1 event store (the SIEM)        │
   │  read API → ops.tbot.trade (behind Access) · /soar console    │
   └──────────────────────────────────────────────────────────────┘
```

## Tech stack

- **Language:** Python 3.13 (engine), TypeScript (Cloudflare Workers)
- **Runtime:** Ubuntu, PM2, cron, nginx, UFW, fail2ban
- **Web:** FastAPI + Uvicorn; vanilla HTML/CSS/JS
- **Data:** append-only JSONL logs with advisory locks; Cloudflare D1 for events
- **AI:** Anthropic Claude, measured per lane against its own cost
- **Venues:** Kalshi (live); OANDA, IBKR and Kraken integrations built and retired
- **External data:** NOAA / National Weather Service
- **Delivery:** push to `main` → a CI gate (kill-switch tests, red-team self-check, secret scan) → deploy in about 40 seconds

## What I'd build next

1. **A numeric forecast ensemble** (NBM, GEFS, ECMWF) calibrated on settlements, instead of a larger language model: the measured lever is forecast skill, not model size.
2. **Market breadth**: more cities at capped size, so position size can grow without moving the price.
3. **Strategy replay for prospective subscribers**: "if you had subscribed 30 days ago", from real fills.

## Contact

Code is private. The [subscriber client](https://github.com/kenmwara/tbot-client) is open source. If you are hiring and want to talk through the design decisions: [linkedin.com/in/kenmwara](https://linkedin.com/in/kenmwara).
