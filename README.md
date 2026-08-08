# BIP-110 Fork Monitor

Real-time Bitcoin fork monitor comparing **Bitcoin Core** vs **Bitcoin Knots (RDTS/BIP-110)**.

🌐 **Live:** https://bip110-fork-monitor.surge.sh

## Data Sources

- **Bitcoin Core chain:** [mempool.space API](https://mempool.space/docs/api)
- **Bitcoin Knots chain:** [bip110.run](https://bip110.run) (runs a real Knots 29.2.0 node)
- **Node census:** [luke.dashjr.org](https://luke.dashjr.org/programs/bitcoin/files/charts/historical.html)

## Features

- Live block height, hash, mining pool, tx count (refreshes every 30s)
- BIP-110 signaling detection via version bit 4
- Current difficulty period signaling stats
- Per-block OP_RETURN carrier analysis from real Knots node
- Fork topology visualization (Core vs Knots chain split detection)
- Network node census (Core vs Knots vs RDTS distribution)
- Sound on new block (opt-in)

## Tech

Single-file vanilla HTML/CSS/JS. No build step, no framework, no backend. Just fetch calls to public APIs.

Built by **Centauri** ⭐
