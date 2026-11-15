# DXM (Dexhune Markets) - proof of concept

A single-file, browser-based client for a decentralized price-oracle network. Open `dxm-v2.html` from disk (`file://`). It uses Alpine.js and ethers.js (loaded from jsDelivr, so the first load needs internet) and stores everything in IndexedDB.

Landing on the page creates a node identity automatically and shows a CoinGecko-style markets view. Everything else (wallet, registry, feeds, proposals) lives in the **Options** drawer.

> **Status: proof of concept.** This is the local half of a node. There is no peer-to-peer layer yet, so instances do not talk to each other. See [Status](#status).

## Status

| Area | State |
| --- | --- |
| Headless wallet and node identity | Working |
| Seed encryption (password) | Working |
| Pool reads (Uniswap v2/v3), pinned to one block | Working, untested against live RPCs |
| Aggregation (USDT conversion, outlier rejection, liquidity weighting) | Working, untested against live RPCs |
| EIP-712 signed reports and weighted-quorum verification | Working locally |
| Price log and charts | Working (history starts when logging starts) |
| Registry manifest (stand-in for the Market DAO) | Local JSON |
| Feed proposals and signed votes | Working locally, not shared with anyone |
| Markets table | Sample data until live feeds are added |
| Peer transport, consensus rounds, peer-set discovery | **Not built** |
| Age and work-done witnessing | **Not built** (currently local claims only) |
| Batching (export/import of data) | **Not built** |
| On-chain verifier contract, real DAO | **Not built** |

## Running it

1. Open `dxm-v2.html` in a Chromium-based browser (other browsers should work for most features, but are untested).
2. A phrase and node key are generated on first load. Open **Options > Wallet** to reveal the phrase and set a password.
3. To get a live feed: **Options > Registry** (optional: register your node as a manager for local testing), then **Feeds and proposals**, submit a proposal, and vote. Once the yes weight and signer count meet the manifest, the feed is added to the markets list. Click its row and press **Fetch prices**.
4. Only Ethereum is configured. Add chains in the manifest with an RPC, USDT and wrapped-native addresses, and a base WNATIVE/USDT pool. **Verify the default pool and token addresses yourself before relying on any price.**

## How it works

### Wallet and keys
- On first load a BIP-39 phrase is generated and account 0 (`m/44'/60'/0'/0/0`) becomes the node key.
- Setting a password encrypts the phrase (PBKDF2-SHA256, 600k iterations, AES-GCM). Unlocking lets the user derive more accounts and choose a different node key.
- **The selected node key is stored unencrypted** in IndexedDB so the node can sign without prompting. The password protects the phrase only. This is an accepted trade-off for the proof of concept. Chrome shares one origin across all local `file://` pages, so other local HTML files could read it. Do not hold funds on this key.

### Price collection
For each feed (token address plus a list of pools, each `v2` or `v3`):
1. Pick one block and read every pool at that block.
2. Match the intended token by **address** against `token0` and `token1`. Pools that do not contain it are rejected.
3. Compute each pool's price with BigInt math (v2 from reserves, v3 from `sqrtPriceX96`) and its liquidity (v3 uses virtual reserves from `liquidity()`).
4. Convert to USDT directly (USDT-quoted pools) or through the chain's base WNATIVE/USDT pool.
5. Drop pools further than 5% from the median, ignore pools under $1,000 of liquidity, and publish a liquidity-weighted average (the simple average is shown too).

Prices are spot prices. Fee-on-transfer and rebasing tokens are not handled, and RPC trust is out of scope for the proof of concept.

### Signed reports
EIP-712, domain `DXM Oracle` v1, type `Report(feedId bytes32, chainId uint256, price uint256, timestamp uint256, seq uint256, epoch uint256, block uint256)`. Price is fixed-point with 18 decimals. `seq` increases per feed.

### Weighted quorum
`verifyReport` recovers each signer, looks up its weight in the manifest, counts each signer once, and accepts when **distinct signers >= `minSigners`** and **total weight >= `threshold`**. Weights are deliberately **uncapped**: a heavy node can carry a quorum together with enough other signers. This is a known property of the design, not a bug.

### Registry manifest (stand-in for the Market DAO)
JSON with `epoch`, `minSigners`, `threshold`, `managers` (address to weight) and `chains`. In the full design the DAO holds this, weights come from age, size and work done, the DAO prevents forks, and its integrity rests on the first nodes. The manifest has the same shape the on-chain registry is meant to expose.

### Feeds and proposals
A feed is an address plus an exchange type. Only nodes can propose feeds. Proposals collect signed votes (`signMessage("DXM vote <id> <yes|no>")`) and are accepted by the same weight-and-signer rule as reports. In this version everything is local to the browser.

### Storage
IndexedDB database `dxm`: a key-value store (wallet, identity, manifest, feeds, proposals, node birth time) and a `prices` log. IndexedDB can be evicted by the browser, and moving the file can orphan the data. The planned batching feature would address this.

## Known limitations
- Not a network node yet: no peers, no consensus, no shared state.
- Age and work done are self-reported. They only mean something once peers witness and agree on them.
- Votes by keys that are not in the manifest are recorded with weight 0.
- Browser tabs are not reliable always-on nodes. This is accepted for the proof of concept (keep-alive tricks aside); other node implementations can come later.
- Not audited. Do not point real value at it.

## Roadmap
1. Peer transport (WebRTC/WebSocket) with bootstrap from manager endpoints listed in the registry, plus peer exchange.
2. Consensus round: pinned block, signed observations, weighted median, signed final report.
3. Age and work-done witnessing and weight calculation.
4. Batching: automatic exports to a chosen directory and refetching missing batches.
5. More exchange adapters, more chains, real volume and market-cap data, chart backfill from chain data.
6. Network setup UI and testnet deployment.
7. On-chain pull-oracle verifier contract and the real Market DAO.
8. Possible later: join fees (kept as a code comment, not a feature), decentralized RPC over the hybrid line.
