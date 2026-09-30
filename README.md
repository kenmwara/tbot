# T BOT — autonomous trading platform

> A live-money trading system that has to earn every lane it runs: 17 logged risk gates, books reconciled to the exchange every night, and its own detection-and-response stack.

🌐 **Live demo:** [tbot.trade/demo](https://tbot.trade/demo)
🛡️ **Security operations:** [github.com/kenmwara/tbot-security](https://github.com/kenmwara/tbot-security)
📦 **Subscriber client:** [github.com/kenmwara/tbot-client](https://github.com/kenmwara/tbot-client) *(open source, the original signal-feed model)*
🔒 **Operator-side bot:** private (strategy IP)

---

## Screenshots

<p>
  <a href="https://tbot.trade/demo"><img src="https://tbot.trade/portfolio/img/tbot-demo-card.png?v=2" width="400" alt="tbot.trade/demo: the real T BOT operator dashboard running on synthetic data"></a>
  &nbsp;&nbsp;
  <img src="https://tbot.trade/portfolio/img/tbot-soar.jpg" width="400" alt="ops.tbot.trade/soar in demo mode: the security console with verdict, posture tiles, incident timeline and event feed">
</p>

*Left: the real operator dashboard, live at tbot.trade/demo on synthetic data. Right: the security console in its synthetic demo mode.*

<p>
  <a href="https://github.com/kenmwara/tbot-security"><img src="https://raw.githubusercontent.com/kenmwara/tbot-security/main/docs/soar-map.png" width="820" alt="SOAR attack map: 24 hours of traffic the server refused, by country, with the SSH, firewall, web and fail2ban split and the top network per country (real counts, no addresses)"></a>
</p>

*The console's attack map with real counts: 24 hours of traffic the server refused, by country. No addresses are shown, and countries come from an offline table on the server.*

## What this is

T BOT trades on its own, around the clock, from one Ubuntu droplet. One rules-based strategy trades live; the method stays private.

It is built as a multi-lane platform: forex (OANDA), US equities (IBKR), crypto (Kraken) and longer-dated macro positions each have a working integration in the same engine and their own row on the dashboard. **A lane goes live only when its record, net of fees and AI spend, earns it**, so capital sits where the evidence is.

A copy-trade pilot is in private beta: a subscriber connects their own exchange account by API key (encrypted at rest, AES-GCM, revocable at any time), and the engine mirrors the operator's trades at proportional size. The subscriber keeps custody. Commission is 15% of realised profit, billed through Stripe. The financial-services app listing is held for regulatory counsel rather than shipped and argued about later.

## What's interesting about it

### 1. A lane pays for itself, or it is recalibrated

Every lane is billed nightly for its real exchange fees **plus the Claude spend that made its decisions**. When the bill exceeds the return, the lane is flagged for recalibration. The rule does not switch anything off by itself, because that is a risk decision a human makes. An AI-driven lane is measured against its own inference cost like any other: the model is only worth its price if the decisions it makes are.

### 2. Seventeen risk gates, every refusal logged

Every candidate market passes seventeen gates before an order exists: the market type and timing, spread and volume, a blacklist, never touching another lane's position, checks on verified outcomes, per-cycle, open-position and pool exposure caps, a robust-Kelly sizing floor, a fee-aware EV check and an exposure ladder the strategy has to earn. Each refusal is written to a shadow log with the gate's name, so how often each gate fires is measured, not claimed. The gate list in the docs is generated from the code, and a gate added without a description fails the deploy. Placement adds its own breakers on top: the kill switch, a daily-loss halt and a cash guard. Most candidates are refused. The system is tuned for selectivity, not volume.

### 3. Every knob is replayed before it ships

A proposed change to a risk limit is first replayed against logged history and graded on real settlements, with the count, win rate and expected value reported. One proposed loosening would have opened 67 markets in a week at a net loss, so it was refused. A new edge starts at capped size with a scheduled verdict date. When a single bad data reading once triggered a "certain" lock, the fix (two consecutive readings) was replayed first: it kept 59 calls with zero losses and refused 18, all of which would have won or were still pending.

### 4. The books come from the exchange

A nightly job pulls every fill, settlement, deposit and withdrawal from the exchange's own API, and the result has to close to the cash the exchange reports, within $5. Getting it to close taught three things the documentation had wrong: some purchases are reported under a different action than the one placed; there is no settlement fee (so two of the three in-repo fee estimators were off, one at half and one at 1.6×); and opposite positions net into cash the moment they form. It also overturned an earlier documented lifetime figure. The exchange's number is the only one that counts now.

### 5. Security is its own system

Intrusion detection watches for unknown SSH keys, file changes, new ports, and, most usefully, **orders filled on the exchange that the bot never logged**. That is how a misused API key would show, and it engages the kill switch automatically. Every event lands in a SIEM, a SOAR console holds the response playbooks, and a red team attacks the system from GitHub Actions every night. Full write-up: [tbot-security](https://github.com/kenmwara/tbot-security).

### 6. Rebuildable by design

In August 2026 the droplet was destroyed with no snapshot. It was rebuilt from git and the secrets vault to live trading in one evening. Backups are now encrypted nightly and the restore is proven every night rather than assumed.

## Architecture

```
                         DigitalOcean droplet (Ubuntu)
   ┌──────────────────────────────────────────────────────────────┐
   │  st-bot-loop    scan → research → decide → guard → execute   │
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
- **Venues:** one exchange (live); OANDA, IBKR and Kraken integrations built into the same engine
- **Delivery:** push to `main` → a CI gate (kill-switch tests, red-team self-check, secret scan) → deploy in about 40 seconds

## What I'd build next

1. **A calibrated ensemble of numeric models** graded on settlements, instead of a larger language model: the measured lever is prediction skill, not model size.
2. **Market breadth**: more markets at capped size, so position size can grow without moving the price.
3. **Strategy replay for prospective subscribers**: "if you had subscribed 30 days ago", from real fills.

## Contact

Code is private. The [subscriber client](https://github.com/kenmwara/tbot-client) is open source. If you are hiring and want to talk through the design decisions: [linkedin.com/in/kenmwara](https://linkedin.com/in/kenmwara).
