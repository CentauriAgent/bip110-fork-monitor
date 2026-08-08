# BIP-110 & Bitcoin Knots — API & Data Source Research

**Researched:** 2026-08-08  
**Purpose:** Identify all public APIs, block explorers, and data services for BIP-110 signaling and Bitcoin Knots chain data

---

## Summary

| # | Source | Status | Public API? | Knots vs Core? |
|---|--------|--------|-------------|----------------|
| 1 | **bip110monitor.com** | ✅ Live | ✅ Yes (JSON) | ❌ No (Core only, via mempool.space) |
| 2 | **api.bitcoin-data.com** (bgeometrics) | ✅ Live | ✅ Yes (JSON, rate-limited) | ❌ No (daily aggregate) |
| 3 | **bip110.run** | ✅ Live | ✅ Yes (JSON files + REST) | ✅ **YES** — runs Bitcoin Knots node |
| 4 | **forkwatch.tv** | ✅ Live | ❌ No public API | ✅ **YES** — runs Core + Knots nodes |
| 5 | **bip110.org/monitor** | ✅ Live | ❌ Frontend only (fetches bip110monitor.com) | ❌ No |
| 6 | **bipmonitor.com** | ✅ Live | ❌ Frontend only (fetches bip110monitor.com + bip119monitor.com) | ❌ No |
| 7 | **fork.observer** | ❌ 500 Error | N/A | Unknown |
| 8 | **wickedsmartbitcoin.com** | ✅ Live | ❌ Dashboard iframe | ❌ No |
| 9 | **miningpool.observer** | ✅ Live | ❌ No BIP-110 data | ❌ No |
| 10 | **bitcoinknots.org** | ✅ Live | ❌ No API | N/A |
| 11 | **Luke Dashjr node census** | ✅ Live | ✅ Yes (raw data files) | ✅ Yes (Knots/Core/RDTS breakdown) |
| 12 | **mempool.space API** | ✅ Live | ✅ Yes | ❌ No (Core node) |

---

## 1. bip110monitor.com — BEST PUBLIC API

**Status:** ✅ Live, no auth, fast response  
**Data source:** Fetches from mempool.space API (Bitcoin Core node)  
**Chain:** Bitcoin Core (NOT Knots)

### Base URL
```
https://bip110monitor.com/api
```

### Example Response (truncated)
```json
{
  "bip": "110",
  "tip": 961632,
  "chainTip": 961632,
  "periodNum": 477,
  "periodStart": 961632,
  "periodEnd": 963647,
  "totalBlocks": 1,
  "signalingCount": 0,
  "pct": 0,
  "synced": true,
  "updatedAt": "2026-08-08T19:37:50.142Z",
  "periods": [
    {
      "periodNum": 465,
      "startBlock": 937440,
      "endBlock": 939455,
      "signalingCount": 1,
      "totalBlocks": 2016,
      "pct": 0.05
    },
    ...
    {
      "periodNum": 476,
      "startBlock": 959616,
      "endBlock": 961631,
      "signalingCount": 51,
      "totalBlocks": 2016,
      "pct": 2.53
    }
  ]
}
```

### What It Returns
- Current chain tip height
- Current difficulty adjustment period number, start/end blocks
- Signaling count and percentage for current period
- Historical data for all past BIP-110 signaling periods (period 465 onward)
- Whether data is synced

### Authentication
None required.

### Limitations
- ❌ Does NOT run Bitcoin Knots — uses mempool.space (Bitcoin Core)
- ❌ Cannot show Knots chain tip vs Core chain tip
- ❌ Cannot detect chain splits or orphans from Knots perspective
- Data checks `nVersion` field bit 4 for signaling

### curl Example
```bash
curl -sS 'https://bip110monitor.com/api' | jq .
```

---

## 2. api.bitcoin-data.com (bgeometrics) — Daily Signaling

**Status:** ✅ Live, no auth, rate-limited (10 req/hour on free tier)  
**Data source:** Bitcoin Core node  
**Chain:** Bitcoin Core (NOT Knots)

### Endpoints

#### BIP-110 Daily Signaling
```
GET https://api.bitcoin-data.com/v1/bip-110-day
```

Returns daily aggregate of total blocks and BIP-110 signaling blocks.

**Example response (truncated):**
```json
[
  {"theDate":"2026-05-01","totalBlocks":158,"bip110Blocks":0},
  {"theDate":"2026-05-21","totalBlocks":127,"bip110Blocks":1},
  ...
  {"theDate":"2026-08-07","totalBlocks":158,"bip110Blocks":7}
]
```

#### Status/Health
```
GET https://api.bitcoin-data.com/v1/status
```

Returns service health (no BIP-110 specific data, but confirms API is operational).

### Authentication
None, but rate-limited: **10 requests/hour** on free plan.

### Limitations
- ❌ Daily aggregates only (not per-block)
- ❌ Does NOT run Bitcoin Knots
- ❌ Cannot show Knots chain tip vs Core chain tip
- Rate limited
- Other endpoints (bip-110-period, bip-110-blocks, etc.) all return 404

### curl Example
```bash
curl -sS 'https://api.bitcoin-data.com/v1/bip-110-day' | jq .
```

---

## 3. bip110.run — RICHEST DATA SOURCE (runs Knots!)

**Status:** ✅ Live, no auth  
**Data source:** **Bitcoin Knots node** (`umbrel.local bitcoind cookie RPC :9332`)  
**Chain:** Bitcoin Knots ✅

This is the most valuable data source because it actually runs a Bitcoin Knots node and provides rich, detailed signaling data.

### Base URL
```
https://bip110.run/data/
https://bip110.run/api/v1/
```

### Endpoints

#### Signal Map (PRIMARY DATA)
```
GET https://bip110.run/data/signal-map.json
```

**Returns:**
- `source`: "umbrel.local bitcoind cookie RPC :9332 (Bitcoin Knots)" ← **KNOTS NODE!**
- Block range (`from`, `to`)
- Total block count and signaling count
- `bitsBase64`: Full bitfield array (base64-encoded) for every block
- `signalingRecords`: Per-block signaling data with miner attribution:
  - Block height, timestamp, pool name, individual miner names
- `recentSignaling`: Last ~40 signaling blocks with pool/miner detail
- `heroes`: Named miners ranked by signal count and share %
- `chainClock`: Current tip height, period boundaries, difficulty
- `bit4HashpowerEstimate`: Estimated hashpower signaling (with 95% confidence interval)

**Example (truncated):**
```json
{
  "source": "umbrel.local bitcoind cookie RPC :9332 (Bitcoin Knots)",
  "from": 938903,
  "to": 961631,
  "count": 22729,
  "signalingCount": 135,
  "signalingRecords": [
    {"h":961536,"ts":1786165600,"pool":"OCEAN","miners":["OCEANXYZ","Roughnecks"]},
    {"h":961530,"ts":1786162705,"pool":"OCEAN","miners":["OCEANXYZ","Sympatheia"]},
    ...
  ],
  "heroes": [
    {"name":"Roughnecks","n":56,"share":41.5},
    {"name":"SoV","n":35,"share":25.9},
    {"name":"BIP110","n":9,"share":6.7},
    ...
  ],
  "chainClock": {
    "tipHeight": 961631,
    "periodStartHeight": 959616,
    "periodBlocks": 2016,
    "difficulty": 126231507121868.2
  },
  "bit4HashpowerEstimate": {
    "signalingBlocks": 25,
    "signalingSharePct": 2.48,
    "networkHashesPerSecond": 9.12e+20,
    "impliedHashesPerSecond": 2.26e+19,
    "phase": "current-voluntary-window"
  }
}
```

#### Node History (Node Census)
```
GET https://bip110.run/data/node-history.json
```

**Returns:** Luke Dashjr's node census data showing Knots/Core/RDTS node counts over time:
```json
{
  "source": "luke.dashjr.org node census (history.txt + uainfo.json)",
  "latest": {
    "total": 114175,
    "listening": 5108,
    "knots": 23703,
    "core30": 31918,
    "rdts": 17699
  }
}
```

#### Carrier Map
```
GET https://bip110.run/data/carrier-map.json
```

Returns block data with detailed per-block carrier/payload analysis.

#### Block Detail API
```
GET https://bip110.run/api/v1/block/{height}
GET https://bip110.run/api/v1/blocks/
```

Returns detailed per-block data including OP_RETURN payload classification, carrier analysis, and signaling status.

#### MSTR BTC Live
```
GET https://bip110.run/api/mstr-btcusd-live
```

MicroStrategy / BTC price data (not BIP-110 related).

### Authentication
None required.

### Strengths
- ✅ **Runs Bitcoin Knots** — actual Knots chain tip
- ✅ Per-block signaling records with miner attribution
- ✅ Hashpower estimates with confidence intervals
- ✅ Node census data (Knots vs Core adoption)
- ✅ Detailed payload/carrier analysis per block
- ✅ Real-time data (generated_utc timestamp)

### Limitations
- No explicit Core-vs-Knots chain comparison endpoint
- `signal-map.json` is a static file (cached, not real-time WebSocket)
- No documented API spec — endpoints discovered from frontend source

### curl Examples
```bash
# Primary signaling data
curl -sS 'https://bip110.run/data/signal-map.json' | jq .

# Node census
curl -sS 'https://bip110.run/data/node-history.json' | jq .

# Carrier/payload map
curl -sS 'https://bip110.run/data/carrier-map.json' | jq . | head -50

# Specific block detail
curl -sS 'https://bip110.run/api/v1/block/961536' | jq .
```

---

## 4. forkwatch.tv — Dual Node Visualizer (Core + Knots)

**Status:** ✅ Live, no public API  
**Data source:** Two independent nodes (Bitcoin Core + Bitcoin Knots), Tor-only, blocks-only  
**Chain:** Both Core AND Knots ✅

Forkwatch runs the most sophisticated comparison setup: two pruned nodes syncing independently over Tor, comparing block-by-block. However, it has **no public API** — data is only accessible through the web frontend (React SPA).

### What It Does
- Runs Bitcoin Core v31.1 and Bitcoin Knots v29.3.knots20260508 side-by-side
- Compares every block through both nodes' validation rules
- Shows "WOULD VIOLATE" verdicts on blocks that Knots rejects but Core accepts
- Countdown timer to height 961,632 (mandatory signaling window start)
- Isometric chain visualization showing Core vs Knots fork points

### Architecture (from GitHub README)
- Backend: Rust/axum + SQLite
- Frontend: React/Tailwind
- Nodes: Tor-only, blocks-only, pruned, bootstrapped from assumeutxo at height 840,000
- WebSocket live updates (but no documented public API)

### GitHub
```
https://github.com/orangeshyguy21/forkwatch
```

### API Status
- ❌ No public REST API
- ❌ No documented endpoints
- Has internal WebSocket for frontend updates
- Would need to reverse-engineer the WS protocol or self-host

### Limitations
- No API access
- Frontend is a JS SPA — data not easily scrapable
- Self-hosted on LAN at forkwatch.local (public site is read-only demo)

---

## 5. bip110.org/monitor

**Status:** ✅ Live  
**Data source:** Fetches from bip110monitor.com API  
**Chain:** Bitcoin Core (via bip110monitor.com)

A nice Astro-built frontend for the bip110monitor.com API. No independent data source.

### API
No independent API. Uses `bip110monitor.com/api` under the hood.

---

## 6. bipmonitor.com

**Status:** ✅ Live  
**Data source:** Fetches from bip110monitor.com and bip119monitor.com APIs  
**Chain:** Bitcoin Core

Landing page linking to BIP-110 and BIP-119 monitors. No independent data.

---

## 7. fork.observer

**Status:** ❌ 500 Internal Server Error (nginx)  
**Data source:** Unknown  
**API:** Unknown

Site is currently down. Previously may have provided fork monitoring data. Will need rechecking later.

---

## 8. wickedsmartbitcoin.com/bip110_signaling

**Status:** ✅ Live  
**Data source:** Self-contained dashboard (iframe)  
**Chain:** Unknown (likely Core)

An elaborate dashboard with visualizations. Data loads inside an iframe (`webapps/bip110_signaling/dashboard.html`). No easily accessible API.

---

## 9. miningpool.observer

**Status:** ✅ Live  
**Data source:** Independent  
**Chain:** Bitcoin Core

Focused on mining pool transaction selection transparency (templates vs actual blocks, missing/conflicting transactions, sanctioned transactions). **No BIP-110 specific data.**

---

## 10. bitcoinknots.org

**Status:** ✅ Live  
**Chain:** N/A (website)

Official Bitcoin Knots website. Provides download links, news, security info. No API, no block explorer, no public RPC endpoints.

**Latest version:** 29.4.knots20260508

---

## 11. Luke Dashjr Node Census (used by bip110.run)

**Status:** ✅ Live  
**Data source:** Network crawler  

Raw data files showing Bitcoin node software distribution including Knots, Core, and RDTS counts.

### URLs
```
https://luke.dashjr.org/programs/bitcoin/files/charts/data/history.txt
https://luke.dashjr.org/programs/bitcoin/files/charts/data/uainfo.json
https://luke.dashjr.org/programs/bitcoin/files/charts/software.html
https://luke.dashjr.org/programs/bitcoin/files/charts/historical.html
```

### Current Numbers (as of 2026-08-08)
- Total nodes: ~116,030
- Listening: 5,206
- Bitcoin Knots v29 (listening): 923
- Bitcoin Knots v29 (non-listening): 21,620
- RDTS/BIP-110 nodes: ~17,990

### curl Example
```bash
curl -sS 'https://luke.dashjr.org/programs/bitcoin/files/charts/data/uainfo.json' | jq . | head -50
```

---

## 12. mempool.space API

**Status:** ✅ Live, no auth  
**Data source:** Bitcoin Core node  
**Chain:** Bitcoin Core

Standard block explorer API. Can be used to check block version fields for BIP-110 signaling detection.

### Example
```bash
# Get recent blocks with version fields
curl -sS 'https://mempool.space/api/v1/blocks' | jq '.[].version'

# Version 537919488 = 0x20080010 → bit 4 set = BIP-110 signaling
# Version 536870912 = 0x20000000 → no signaling bits set
```

### BIP-110 Detection Logic
BIP-110 uses bit 4 (value 0x20000010 in the version field, with BIP9 prefix 0x20000000):
- Block version & 0x1FFFFFFF must equal 0x20000010 for signaling
- Or more precisely: `(version >> 0) & 0x1FFFFFFF` has top 3 bits as `001` (BIP9 prefix) and bit 4 set

---

## Bitcoin Knots Public RPC / Electrum Servers

### Finding: NONE FOUND

There are **no known public Bitcoin Knots RPC endpoints or Electrum servers**. Key findings:

1. **bitcoinknots.org** does not list any public nodes, RPC endpoints, or electrum servers
2. **No "Knots block explorer" exists** — all known explorers (mempool.space, blockstream.info, etc.) run Bitcoin Core
3. **bip110.run** is the closest thing — it exposes data FROM a Knots node via JSON files, but not the RPC itself
4. **forkwatch.tv** runs Knots but doesn't expose a public API
5. Bitcoin Knots is designed for self-hosted use — public endpoints would be unusual

### Implications
To get real Knots chain data, you must either:
1. **Run your own Bitcoin Knots node** (recommended)
2. **Use bip110.run's cached data** (JSON files from their Knots node)
3. **Self-host forkwatch** (Core + Knots comparison)

---

## Recommendations for Fork Monitor Project

### Best Data Sources to Use

1. **bip110.run** (`/data/signal-map.json`) — PRIMARY
   - Runs actual Knots node
   - Rich per-block data with miner attribution
   - Hashpower estimates
   - No auth, no rate limit observed

2. **bip110monitor.com** (`/api`) — SECONDARY
   - Clean JSON API
   - Historical period data
   - Good for aggregate trends

3. **api.bitcoin-data.com** (`/v1/bip-110-day`) — DAILY TRENDS
   - Daily signaling counts
   - Good for charting long-term trends

4. **Luke Dashjr census** — NODE COUNTS
   - Knots/Core adoption metrics
   - Via bip110.run's processed version or raw files

5. **mempool.space** — BLOCK DATA
   - Detailed block/transaction data
   - Core perspective for comparison

### What's Missing (gaps our tool could fill)
- **No public Core-vs-Knots chain comparison API** — forkwatch.tv does this internally but has no API
- **No real-time alerting** when Knots rejects a block that Core accepts
- **No unified dashboard** combining signaling data + node census + chain comparison
- **fork.observer is down** — opportunity to fill the gap

### Suggested Polling Strategy
```
Every 10 min:
  - GET bip110.run/data/signal-map.json (Knots perspective, signaling)
  - GET bip110monitor.com/api (aggregate signaling stats)

Every 1 hour:
  - GET api.bitcoin-data.com/v1/bip-110-day (daily trend update)

On new block (via mempool.space WebSocket):
  - Check block version for bit 4 signaling
  - Compare with Knots node data if available
```

---

## Appendix: All Discovered Endpoints

| Endpoint | Method | Auth | Returns |
|----------|--------|------|---------|
| `bip110monitor.com/api` | GET | None | Current + historical BIP-110 signaling periods |
| `api.bitcoin-data.com/v1/bip-110-day` | GET | Rate-limited | Daily BIP-110 block counts |
| `api.bitcoin-data.com/v1/status` | GET | Rate-limited | Service health |
| `bip110.run/data/signal-map.json` | GET | None | Full signaling map from Knots node |
| `bip110.run/data/node-history.json` | GET | None | Node census data (Knots/Core/RDTS) |
| `bip110.run/data/carrier-map.json` | GET | None | Per-block carrier/payload analysis |
| `bip110.run/api/v1/block/{height}` | GET | None | Detailed block analysis |
| `bip110.run/api/v1/blocks/` | GET | None | Block list |
| `bip110.run/api/mstr-btcusd-live` | GET | None | MSTR/BTC price (non-BIP) |
| `luke.dashjr.org/.../data/history.txt` | GET | None | Raw node census history |
| `luke.dashjr.org/.../data/uainfo.json` | GET | None | Current node software distribution |
| `mempool.space/api/v1/blocks` | GET | None | Recent blocks (Core perspective) |
| `mempool.space/api/v1/blocks/tip/hash` | GET | None | Current Core chain tip hash |
