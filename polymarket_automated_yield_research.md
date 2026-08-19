# Polymarket: Automated Yield Generation Research
## Comprehensive Technical & Strategic Analysis
**Date: March 10, 2026**

---

# Table of Contents

1. [What is Polymarket](#1-what-is-polymarket)
2. [API & Infrastructure](#2-api--infrastructure)
3. [Automated Strategies](#3-automated-strategies)
4. [Edge Analysis — Where Alpha Exists](#4-edge-analysis--where-alpha-exists)
5. [Technical Details — Blockchain & Settlement](#5-technical-details--blockchain--settlement)
6. [Risks](#6-risks)
7. [Fees](#7-fees)
8. [Open-Source Tools & Repositories](#8-open-source-tools--repositories)
9. [Sources](#9-sources)

---

# 1. What is Polymarket

## 1.1 Overview

Polymarket is a decentralized prediction market platform launched in 2020 on the Polygon blockchain. Traders buy and sell shares representing outcomes of real-world events using USDC. Shares are priced between $0.00 and $1.00, where the price directly represents the market's implied probability for that outcome. There is no centralized "house" — it is purely peer-to-peer matching of buyers and sellers.

**Key Stats (as of early 2026):**
- Largest prediction market by volume globally
- Secured CFTC Designated Contract Market (DCM) approval in November 2025
- Acquired QCEX (CFTC-licensed exchange and clearinghouse) for $112M in July 2025
- Now operates both a global (offshore) and a US-regulated platform

## 1.2 Central Limit Order Book (CLOB)

Polymarket uses a **hybrid-decentralized CLOB** rather than an AMM:

- **Off-chain matching**: An operator collects orders in a central order book and matches them off-chain for speed and efficiency
- **On-chain settlement**: Matched trades settle atomically on Polygon via the Exchange smart contract using EIP-712 signed order messages
- **Non-custodial**: The operator cannot set prices or execute unauthorized trades. Users retain custody of their funds via smart contracts
- **No front-running**: The hybrid model prevents MEV extraction common in pure on-chain AMMs

This architecture combines the efficiency of centralized exchange matching with the security guarantees of blockchain settlement.

## 1.3 Conditional Token Framework (CTF)

All outcomes are tokenized using the **Conditional Token Framework (CTF)**, an ERC-1155 token standard developed by Gnosis:

- **Minting (Split)**: A user deposits USDC.e, which is locked in the CTF contract. In return, they receive one YES token and one NO token. Every Yes/No pair is backed by exactly $1.00 USDC.e
- **Merging**: Users can reverse the split by returning matched YES + NO pairs to reclaim USDC.e
- **Redemption**: After market resolution, winning tokens are redeemable for $1.00 USDC.e each; losing tokens become worthless

**Token ID Computation:**
1. **Condition ID** = hash(oracle_address, question_hash, outcome_count)
2. **Collection ID** = derived per outcome using an indexSet bitmask (1 for YES, 2 for NO)
3. **Position ID (Token ID)** = hash(collateral_token_address, collection_id)

**Multi-Outcome Markets (Neg Risk):**
Markets with 3+ outcomes use the NegRiskAdapter, which converts collections of NO tokens across mutually-exclusive binary markets into YES token positions. These require `negRisk: true` when placing orders.

## 1.4 Market Resolution

Polymarket primarily uses **UMA's Optimistic Oracle** for resolution:

1. A proposed resolution is submitted
2. There is a challenge period where anyone can dispute the resolution
3. If disputed, UMA token holders vote to determine the correct outcome
4. Winning tokens become redeemable for $1.00 USDC.e

**Vulnerability**: The UMA dispute system has been criticized for potential manipulation by large token holders who can influence votes. The regulated US exchange may use a different resolution mechanism.

---

# 2. API & Infrastructure

## 2.1 API Architecture Overview

Polymarket provides four distinct API services:

| API | Base URL | Auth Required | Purpose |
|-----|----------|--------------|---------|
| **Gamma API** | `https://gamma-api.polymarket.com` | No | Market metadata, events, tags, search, discovery |
| **CLOB API** | `https://clob.polymarket.com` | Partial (trading endpoints) | Order book, prices, order placement/management |
| **Data API** | `https://data-api.polymarket.com` | No | User positions, trades, activity, leaderboards |
| **Bridge API** | `https://bridge.polymarket.com` | Yes | Deposits and withdrawals (proxies fun.xyz) |

## 2.2 Gamma API — Market Discovery

**Base URL**: `https://gamma-api.polymarket.com`

**Key Endpoints:**
- `GET /events` — Fetch events (each event contains associated markets)
- `GET /events/slug/{slug}` — Fetch specific event by slug
- `GET /markets` — Query markets with filtering
- `GET /markets/slug/{slug}` — Fetch specific market by slug
- `GET /sports` — Sports metadata, tag IDs, resolution sources
- `GET /tags` — All available market tags/categories

**Key Parameters:**
| Parameter | Purpose |
|-----------|---------|
| `slug` | Unique identifier from Polymarket URL |
| `tag_id` | Filter by category/sport |
| `active` | Boolean — filter for live tradable markets |
| `closed` | Boolean — filter for closed markets |
| `limit` / `offset` | Pagination |
| `order` | Sort by `volume_24hr`, `liquidity`, `start_date`, etc. |
| `ascending` | Sort direction |
| `related_tags` | Include related tag markets |
| `exclude_tag_id` | Exclude specific tags |

**Response includes**: `clobTokenIds` (needed for pricing and trading), `condition_id`, market metadata, resolution sources.

**Best Practice**: Use `active=true&closed=false` to filter for live markets only.

## 2.3 CLOB API — Trading

**Base URL**: `https://clob.polymarket.com`

### Public Endpoints (No Auth):

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/book` | GET | Order book for a single token |
| `/books` | GET | Batch order books |
| `/price` | GET | Current price for a token on a given side (BUY/SELL) |
| `/prices` | GET | Batch prices |
| `/midpoint` | GET | Midpoint between best bid and ask |
| `/midpoints` | GET | Batch midpoints |
| `/spread` | GET | Current spread |
| `/spreads` | GET | Batch spreads |
| `/prices-history` | GET | Historical price data |
| `/tick-size` | GET | Market tick size |
| `/ok` | GET | Health check |

### Authenticated Endpoints (L2 Auth Required):

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/order` | POST | Place a single order |
| `/orders` | POST | Place batch orders (up to 15) |
| `/order` | DELETE | Cancel a single order |
| `/orders` | DELETE | Cancel batch orders |
| `/cancel-all` | DELETE | Cancel all open orders |
| `/cancel-market-orders` | DELETE | Cancel all orders in a specific market |
| `/orders` | GET | Retrieve open orders |
| `/order` | GET | Retrieve specific order |
| `/trades` | GET | Trade history |
| `/data/orders` | GET | Historical order data |
| `/data/trades` | GET | Historical trade data |
| `/notifications` | GET | Order notifications |
| `/balance-allowance` | GET | Balance and allowance info |
| `/balance-allowance` | POST | Update balance allowance |

### Order Types:

| Type | Code | Behavior |
|------|------|----------|
| **GTC** | Good-Til-Cancelled | Rests on book until filled or cancelled. Default for passive quoting |
| **GTD** | Good-Til-Date | Rests on book until specified UTC timestamp. Min 1 minute from now |
| **FOK** | Fill-Or-Kill | Must fill entirely and immediately, or entire order is cancelled |
| **FAK** | Fill-And-Kill | Fills available quantity immediately, remainder cancelled |
| **Post-Only** | (flag) | Only rests on book; rejected if it would immediately match. GTC/GTD only |

### Order Parameters:

```
token_id: str       — Outcome token ID (from Gamma API)
price: float        — Price per share (0.01 - 0.99)
size: float         — Number of shares (BUY limit) or dollar amount (BUY market)
side: str           — "BUY" or "SELL"
expiration: int     — Unix timestamp (GTD orders)
negRisk: bool       — true for multi-outcome markets
feeRateBps: int     — Fee rate in basis points (for fee-enabled markets)
```

### Tick Sizes:
Markets have configurable minimum price increments: `0.1`, `0.01`, `0.001`, or `0.0001`. Orders that violate the tick size are rejected. Query via the Gamma API `minimum_tick_size` field or the CLOB `/tick-size` endpoint.

## 2.4 Data API — Analytics

**Base URL**: `https://data-api.polymarket.com`

| Endpoint | Purpose |
|----------|---------|
| `/positions` | Current user positions (filter by market/event) |
| `/closed-positions` | Historical closed positions |
| `/trades` | Trade history for a user or market |
| `/activity` | On-chain activity for a wallet |
| `/leaderboard` | Trader leaderboard rankings |
| `/open-interest` | Market open interest |

All Data API endpoints are public (no authentication required).

## 2.5 WebSocket API — Real-Time Data

**Four WebSocket channels:**

| Channel | URL | Auth | Purpose |
|---------|-----|------|---------|
| **Market** | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | No | Orderbook snapshots, price updates, trades, best bid/ask |
| **User** | `wss://ws-subscriptions-clob.polymarket.com/ws/user` | Yes | Personal trade lifecycle, order modifications |
| **Sports** | `wss://sports-api.polymarket.com/ws` | No | Live game scores, event status |
| **RTDS** | `wss://ws-live-data.polymarket.com` | Optional | Real-time data stream |

**Market Channel Subscription:**
```json
{
  "assets_ids": ["token_id_1", "token_id_2"],
  "type": "market",
  "custom_feature_enabled": true
}
```

**User Channel Subscription (Authenticated):**
```json
{
  "auth": {
    "apiKey": "your-api-key",
    "secret": "your-secret",
    "passphrase": "your-passphrase"
  },
  "markets": ["0x1234...condition_id"],
  "type": "user"
}
```

**Dynamic Subscription Updates** (no reconnection needed):
```json
{"assets_ids": ["new_token_id"], "operation": "subscribe"}
```

**Heartbeat Protocol:**
- Market/User: Send `PING` every 10 seconds; server responds `PONG`
- Sports: Server sends `ping` every 5 seconds; respond `pong` within 10 seconds
- **Critical**: If no valid heartbeat in 10 seconds (5s buffer), ALL open orders are cancelled

## 2.6 Authentication

### Two-Tier Authentication Model

**L1 Authentication (Private Key / EIP-712):**
- Used for: Creating API credentials, signing orders
- Mechanism: EIP-712 typed data signature using wallet private key
- Signs a `ClobAuth` struct with address, timestamp, nonce, and attestation message
- Domain chainId: 137 (Polygon mainnet)

**L1 Headers:**
| Header | Purpose |
|--------|---------|
| `POLY_ADDRESS` | Polygon signer address |
| `POLY_SIGNATURE` | EIP-712 signature |
| `POLY_TIMESTAMP` | Current UNIX timestamp |
| `POLY_NONCE` | Nonce (default: 0) |

**L2 Authentication (API Key / HMAC-SHA256):**
- Used for: All trading endpoint requests
- Generated from L1 authentication
- Returns: `apiKey` (UUID), `secret` (base64), `passphrase` (random string)

**L2 Headers (all 5 required for trading endpoints):**
| Header | Source |
|--------|--------|
| `POLY_ADDRESS` | User's wallet address |
| `POLY_SIGNATURE` | HMAC-SHA256 using API secret |
| `POLY_TIMESTAMP` | Current UNIX timestamp |
| `POLY_API_KEY` | Generated apiKey value |
| `POLY_PASSPHRASE` | Generated passphrase value |

**Credential Generation:**
- REST: `POST /auth/api-key` (create new) or `GET /auth/derive-api-key` (retrieve existing)
- Python SDK: `client.create_or_derive_api_creds()`
- TypeScript SDK: `client.createOrDeriveApiKey()`

### Signature Types (Wallet Configurations):

| Type | Value | Use Case |
|------|-------|----------|
| **EOA** | 0 | Standard wallets; user pays gas directly |
| **POLY_PROXY** | 1 | Magic Link / email wallets with exported private keys |
| **GNOSIS_SAFE** | 2 | Browser/embedded wallet accounts (most common) |

The `funder` address is the proxy wallet address displayed on Polymarket.com that holds the trading funds.

## 2.7 SDK Clients

### Python SDK (`py-clob-client`)

```bash
pip install py-clob-client  # Python 3.9+
```

**Initialization:**
```python
from py_clob_client.client import ClobClient

# Read-only (no auth needed)
client = ClobClient("https://clob.polymarket.com")

# Full trading access
client = ClobClient(
    host="https://clob.polymarket.com",
    key="0xYOUR_PRIVATE_KEY",      # Ethereum private key
    chain_id=137,                   # Polygon mainnet
    signature_type=1,               # 0=EOA, 1=POLY_PROXY, 2=GNOSIS_SAFE
    funder="0xYOUR_PROXY_WALLET"   # Proxy wallet address
)

# Generate and set API credentials
creds = client.create_or_derive_api_creds()
client.set_api_creds(creds)
```

**Key Methods:**

| Method | Purpose |
|--------|---------|
| `get_order_book(token_id)` | Get order book (bids/asks) |
| `get_order_books([BookParams(...)])` | Batch order books |
| `get_price(token_id, side)` | Current price for BUY or SELL side |
| `get_midpoint(token_id)` | Midpoint price |
| `get_balance()` | USDC balance (in wei; divide by 1e6) |
| `get_balance_allowance(params)` | Balance and token allowance |
| `get_positions()` | Current positions |
| `get_orders(params)` | Open orders |
| `get_order(order_id)` | Single order details |
| `create_order(OrderArgs(...))` | Sign a limit order locally |
| `create_market_order(MarketOrderArgs(...))` | Sign a market order locally |
| `post_order(signed_order, order_type)` | Submit signed order |
| `post_orders(signed_orders, order_type)` | Batch submit (up to 15 orders) |
| `cancel(order_id)` | Cancel single order |
| `cancel_all()` | Cancel all orders |

**Limit Order Example:**
```python
from py_clob_client.clob_types import OrderArgs, OrderType
from py_clob_client.order_builder.constants import BUY

order_args = OrderArgs(
    token_id="<YES_TOKEN_ID>",
    price=0.50,
    size=100,        # 100 shares
    side=BUY
)
signed_order = client.create_order(order_args)
resp = client.post_order(signed_order, OrderType.GTC)
```

**Market Order Example:**
```python
from py_clob_client.clob_types import MarketOrderArgs, OrderType
from py_clob_client.order_builder.constants import BUY

market_order = MarketOrderArgs(
    token_id="<YES_TOKEN_ID>",
    amount=25.0,     # $25 USDC to spend
    side=BUY,
    order_type=OrderType.FOK
)
signed = client.create_market_order(market_order)
resp = client.post_order(signed, OrderType.FOK)
```

### TypeScript SDK (`@polymarket/clob-client`)

```bash
npm install @polymarket/clob-client
```

```typescript
import { ClobClient, Side, OrderType } from "@polymarket/clob-client";

const client = new ClobClient(
  "https://clob.polymarket.com",
  137,
  wallet,       // ethers.js Wallet or Signer
  undefined,    // creds (optional)
  1,            // signatureType
  "0xFUNDER"   // funder address
);

// Create and post a limit order
const response = await client.createAndPostOrder(
  { tokenID: "TOKEN_ID", price: 0.5, size: 10, side: Side.BUY },
  { tickSize: "0.01", negRisk: false },
  OrderType.GTC
);
```

## 2.8 Rate Limits

### CLOB API Rate Limits:

| Endpoint | Burst (per 10s) | Sustained (per 10m) |
|----------|-----------------|---------------------|
| General | 9,000 | — |
| `POST /order` | 3,500 | 36,000 |
| `DELETE /order` | 3,000 | 30,000 |
| `POST/DELETE /orders` (batch) | 1,000 | 15,000 |
| `DELETE /cancel-all` | 250 | 6,000 |
| `/book`, `/price`, `/midpoint` | 1,500 each | — |
| `/books`, `/prices`, `/midpoints` | 500 each | — |
| `/prices-history` | 1,000 | — |
| `/trades`, `/orders` | 900 | — |

### Gamma API Rate Limits:
| Endpoint | Rate (per 10s) |
|----------|---------------|
| General | 4,000 |
| `/events` | 500 |
| `/markets` | 300 |

### Data API Rate Limits:
| Endpoint | Rate (per 10s) |
|----------|---------------|
| General | 1,000 |
| `/trades` | 200 |
| `/positions` | 150 |

**Throttling behavior**: Requests are throttled (delayed/queued) rather than immediately rejected, using sliding time windows.

---

# 3. Automated Strategies

## 3.1 Market Making (Liquidity Provision)

### Strategy Overview
Market makers simultaneously post both buy (bid) and sell (ask) orders, earning the spread between them. On Polymarket, this means posting orders to buy YES shares slightly below the midpoint and sell YES shares slightly above — capturing profit on each completed round-trip.

### Revenue Sources
1. **Spread capture**: Profit from the bid-ask differential on each round-trip
2. **Liquidity rewards**: Daily USDC payouts from Polymarket's incentive program
3. **Maker rebates**: 20-25% of taker fees returned to makers in fee-enabled markets

### Liquidity Rewards Program
- Rewards traders for placing limit orders within the market's maximum spread (shown as blue lines in the order book)
- **Closer to midpoint = higher rewards**
- **Two-sided orders earn ~3x more** than one-sided positions
- Rewards paid daily at ~midnight UTC
- Minimum payout threshold: $1
- If midpoint is below $0.10, orders on both sides are required to qualify
- Reward rates and max spreads vary by market

### Maker Rebates Program (Fee-Enabled Markets)
- Makers receive a percentage of collected taker fees:
  - **Crypto markets**: 20% rebate
  - **Sports markets (NCAAB, Serie A)**: 25% rebate
- Calculation: pro-rata based on your share of executed maker volume per market
- Paid daily in USDC directly to your wallet

### Implementation Approach

```
1. MARKET SELECTION
   - Scan Gamma API for active markets
   - Filter by: liquidity, volume_24hr, volatility metrics
   - Prefer low-volatility markets for safer MM (e.g., long-dated political events)
   - Calculate historical volatility across 3h, 24h, 7d, 30d windows

2. FAIR VALUE ESTIMATION
   - Use midpoint from CLOB /midpoint endpoint as baseline
   - Adjust based on: own probability model, order book imbalance, recent trade flow
   - Weight by time-to-resolution (prices compress toward 0 or 1 as events approach)

3. SPREAD DETERMINATION
   - Tighter spreads in liquid markets (higher reward, lower per-trade profit)
   - Wider spreads in volatile or illiquid markets (lower reward, higher per-trade profit)
   - Must stay within the market's max spread to qualify for liquidity rewards
   - Dynamic adjustment based on inventory position

4. ORDER PLACEMENT
   - Use batch POST /orders (up to 15 per call) for efficiency
   - Place GTC orders on both sides
   - Use GTD orders when approaching known catalysts (auto-expire)
   - Send heartbeat every 5 seconds to maintain session

5. INVENTORY MANAGEMENT
   - Track net position (YES vs NO exposure)
   - Skew quotes away from accumulated inventory
   - Use merge operations to consolidate matched YES+NO pairs back to USDC
   - Cap maximum position size per market

6. RISK CONTROLS
   - Cancel all orders immediately on: news events, unusual volume, error conditions
   - Set maximum loss per market/day
   - Monitor WebSocket user channel for real-time fill notifications
   - Validate every order against book midpoint; reject statistical outliers
```

### Profitability Benchmarks
- Reference case: $10,000 capital generated ~$200/day (2% daily) during peak liquidity rewards (pre-late 2024)
- Scaled to $700-800/day at peak with larger capital deployment
- Post-2024 election: liquidity rewards decreased significantly; returns compressed
- Current viability depends heavily on market-specific reward rates

## 3.2 Cross-Platform Arbitrage (Polymarket vs Kalshi)

### Strategy Overview
Exploit price discrepancies between prediction market platforms for identical or equivalent events. Academic research documented **over $40 million in arbitrage profits** extracted from Polymarket alone between April 2024 and April 2025.

### Price Divergence Statistics
- Kalshi and Polymarket prices for identical events diverge by **>5 percentage points ~15-20% of the time**
- Divergence most common for politically charged events where participant composition differs
- Polymarket generally leads price discovery; Kalshi often lags by minutes

### Implementation

```
1. MARKET MATCHING
   - Normalize market descriptions across platforms
   - Match by: event type, resolution date, strike conditions
   - Critical: Verify resolution criteria are IDENTICAL (different platforms may interpret
     the same event differently)

2. PRICE MONITORING
   - Stream prices from both Polymarket (WebSocket) and Kalshi (API)
   - Normalize to standard probability format (0-1)
   - Calculate: cost_polymarket_yes + cost_kalshi_no < $1.00 (or vice versa)

3. EXECUTION
   - If combined cost < $1.00 minus fees: execute simultaneously on both platforms
   - Account for: trading fees, gas costs, capital lockup period
   - Minimum spread threshold to cover transaction costs

4. RISK MANAGEMENT
   - Resolution divergence risk: Different platforms may resolve differently
     (e.g., 2024 government shutdown case — Polymarket used OPM announcement,
      Kalshi required actual shutdown >24 hours)
   - Execution risk: Prices may move between placing orders on two platforms
   - Capital lockup: Funds tied until market resolution on both platforms
   - 78% of arb opportunities in low-volume markets fail due to execution inefficiencies
```

### Available Tools
- `pmxt` — unified wrapper (like CCXT but for prediction markets) normalizing data across platforms
- Several open-source Python bots on GitHub (see Section 8)
- One documented case: $764 profit in a single day on BTC-15m markets with $200 initial deposit

## 3.3 Intra-Platform Arbitrage

### Market Rebalancing Arbitrage
Within a single market, if YES + NO prices don't sum to $1.00:
- **If sum < $1.00**: Buy both YES and NO tokens → guaranteed $1.00 payout at resolution
- **If sum > $1.00**: Sell both sides → guaranteed profit

### Combinatorial Arbitrage
Across correlated markets on Polymarket:
- If Event A has multiple markets that are logically linked, price inconsistencies create arbitrage
- Example: "Will X happen before March?" priced at 40%, "Will X happen before June?" priced at 35% — logical impossibility
- Requires graph-based analysis of market relationships

## 3.4 Statistical / Quantitative Trading

### Strategy Overview
Generate alpha by identifying mispricings between market-implied probability (Polymarket price) and a model-estimated true probability.

### Key Finding
Research shows **nearly half of all conditions on Polymarket were mispriced by an average of 40%**, indicating massive opportunity for systematic traders.

### Implementation

```
1. PROBABILITY MODEL
   - Train ML models on: historical prediction market data, news sentiment,
     polling data, economic indicators, sports statistics
   - Output: estimated true probability for each market outcome
   - Use: LLMs for news synthesis, ensemble models for probability estimation

2. SIGNAL GENERATION
   - Compare model probability vs market price
   - Signal = model_probability - market_price
   - Entry threshold: only trade when |signal| > minimum_edge (e.g., 5-10%)

3. POSITION SIZING
   - Modified Kelly Criterion accounting for execution risk
   - Cap positions at 50% of current order book depth to avoid moving the market
   - Maximum position per market as % of portfolio

4. EXECUTION
   - Use limit orders (GTC) slightly inside the spread for better fills
   - For urgent entries: FAK orders with slippage protection
   - Monitor via WebSocket for real-time fill confirmation

5. CONVERGENCE MONITORING
   - Mean-reverting strategy: prices should converge toward model prediction
   - Set take-profit at: model_probability or pre-defined profit target
   - Set stop-loss to limit downside exposure
   - Time-based exit if convergence doesn't occur within expected window
```

## 3.5 Event-Driven Automated Trading

### Strategy Overview
Markets take **3-15 minutes to fully price in breaking news**, with first movers capturing 20-50% of the eventual price movement. Automated systems can react in seconds.

### Implementation

```
1. NEWS MONITORING
   - Real-time feeds: Twitter/X API, news wire services, government press releases
   - NLP/LLM processing to extract: event, affected markets, directional impact
   - Pre-built mapping of news categories to Polymarket market IDs

2. RAPID PRICING
   - On news trigger: estimate new fair value
   - Compare with current market price (via WebSocket feed)
   - If edge > threshold: execute immediately

3. EXECUTION
   - FOK orders for guaranteed fill (all-or-nothing)
   - FAK orders if partial fills are acceptable
   - Set aggressive price limits to protect against stale quotes

4. RISK CONTROLS
   - Maximum capital per news event
   - Cool-down period after execution
   - Verify news source reliability before trading
```

### Target Markets
- Geopolitical events (military actions, sanctions, diplomatic negotiations)
- Economic data releases (CPI, jobs reports, GDP)
- Sports events (injuries, lineup changes, weather)
- Crypto price markets (exchange listings, regulatory actions)

## 3.6 Time-Decay / Convergence Trades

### Strategy Overview
As markets approach resolution, prices compress toward 0 or 1 as uncertainty decreases. This creates predictable dynamics.

### Implementation

```
1. IDENTIFY HIGH-CONVICTION OUTCOMES
   - Markets trading at 85-95% probability with strong fundamental backing
   - Calculate expected return: (1.00 - current_price) * probability_of_being_correct

2. HOLD-TO-RESOLUTION
   - Buy shares in high-probability outcomes at discount to $1.00
   - The 5-15% discount represents the market's time-value and uncertainty premium
   - As resolution approaches, price converges to $1.00 if outcome is correct

3. RISK
   - Binary outcome: if wrong, lose entire position
   - Best for events with very high certainty but non-trivial time to resolution
```

## 3.7 Cross-Market Hedging

### Strategy Overview
Use positions on Polymarket to hedge exposure on other platforms or asset classes.

### Examples
- Long Bitcoin + Buy "BTC below $80K by June" on Polymarket = hedged downside
- Short a stock + Buy "Company X acquires Y" on Polymarket = event hedge
- Polymarket crypto prediction markets can hedge perp/futures positions on Hyperliquid, Binance, etc.

---

# 4. Edge Analysis — Where Alpha Exists

## 4.1 Mispriced Markets (Low-Liquidity Long-Tail Events)

- **Finding**: Nearly half of all Polymarket conditions are mispriced by ~40% on average
- **Why**: Low liquidity markets receive little attention from sophisticated traders
- **How to exploit**: Systematically scan all markets, estimate true probabilities, trade where edge > threshold
- **Risk**: Low liquidity means high slippage and difficulty exiting positions

## 4.2 Correlated Market Inefficiencies

- **Example**: "Will the Fed cut rates in March?" at 60% while "Will the Fed cut rates by June?" at 55% — logically impossible
- **How to exploit**: Build a dependency graph of all active markets; scan for logical contradictions
- **Academic finding**: Combinatorial arbitrage across multiple markets is a well-documented source of profit
- **Tools**: Graph-based algorithms to detect probability inconsistencies across related markets

## 4.3 Cross-Platform Price Divergence

- **Finding**: Polymarket and Kalshi diverge by >5% approximately 15-20% of the time
- **Why**: Different participant bases (crypto-native vs traditional finance), different fee structures, different UX
- **Polymarket leads price discovery** due to higher liquidity; Kalshi lags by minutes
- **Window**: Arbitrage windows typically last seconds to minutes

## 4.4 News-Driven Rapid Repricing

- **Finding**: Markets take 3-15 minutes to fully price breaking news
- **First-mover advantage**: Capture 20-50% of the eventual price movement
- **Automation edge**: Bots can process news and execute in seconds vs human minutes
- **Example**: $400K profit by one trader betting on Venezuelan presidential removal hours before a surprise military operation (though this raised insider trading concerns)

## 4.5 Longshot Bias

- **Finding**: Traders systematically overvalue underdogs/longshots
- **Research**: Betting favorites yields -3.64% avg loss vs betting outsiders at -26.08% avg loss
- **How to exploit**: Systematically sell overpriced longshot outcomes (sell YES tokens in low-probability markets or buy NO tokens)
- **Risk**: Tail events do occur; must diversify across many markets

## 4.6 Behavioral Biases

- **Recency bias**: Traders overweight recent events
- **Anchoring**: Market prices anchored to initial listing prices
- **Herding**: Momentum-driven price movements overshoot fair value
- **How to exploit**: Contrarian strategies when markets exhibit clear behavioral biases

## 4.7 Structural Alpha in Market Making

- **Liquidity rewards + spread capture + maker rebates** create a triple revenue stream
- **Two-sided quoting earns ~3x more rewards** than one-sided
- **Edge**: Automated systems can monitor hundreds of markets simultaneously while manual traders handle a handful
- **Time advantage**: Bots can adjust quotes instantly when conditions change

---

# 5. Technical Details — Blockchain & Settlement

## 5.1 Blockchain Infrastructure

- **Network**: Polygon PoS (EVM-compatible Layer 2)
- **Chain ID**: 137 (mainnet)
- **Collateral**: USDC.e (bridged USDC on Polygon)
- **Token Standard**: ERC-1155 (Conditional Tokens)
- **Gas Token**: POL (formerly MATIC)
- **Typical gas cost**: $0.01-$0.50 per transaction; most trades $0.05-$0.20

## 5.2 Smart Contract Addresses (Polygon)

| Contract | Address |
|----------|---------|
| **CTF Exchange** | `0x4bfb41d5b3570defd03c39a9a4d8de6bd8b8982e` |
| **Neg Risk CTF Exchange** | `0xc5d563a36ae78145c45a50134d48a1215220f80a` |
| **Neg Risk Fee Module** | `0x78769d50be1763ed1ca0d5e878d93f05aabff29e` |
| **UMA CTF Adapter 2** | `0x6A9D222616C90FcA5754cd1333cFD9b7fb6a4F74` |

## 5.3 Settlement Mechanics

1. **Order Signing**: User signs an EIP-712 typed data message with their private key
2. **Off-chain Matching**: The CLOB operator matches compatible orders
3. **On-chain Execution**: Matched trades are submitted to the Exchange contract on Polygon
4. **Atomic Settlement**: The Exchange contract transfers tokens atomically — USDC.e for outcome tokens

**Trade Statuses (post-match):**
| Status | Terminal | Description |
|--------|----------|-------------|
| MATCHED | No | Sent to executor for on-chain submission |
| MINED | No | Observed on-chain without finality confirmation |
| CONFIRMED | Yes | Strong probabilistic finality — settlement complete |
| RETRYING | No | Transaction failed; operator resubmitting |
| FAILED | Yes | Permanent failure; no retry |

On-chain settlement emits an `OrderFilled` event containing: order hash, maker/taker addresses, asset IDs, filled amounts, and fees.

## 5.4 Token Operations

| Operation | Description | Gas Cost |
|-----------|-------------|----------|
| **Split** | Deposit USDC.e → Receive YES + NO tokens | ~$0.05-0.20 |
| **Merge** | Return YES + NO pair → Receive USDC.e | ~$0.05-0.20 |
| **Redeem** | After resolution, exchange winning tokens for $1.00 USDC.e | ~$0.05-0.20 |
| **Approve** | Grant allowance for exchange to transfer tokens | ~$0.01-0.05 |

**Gasless Transactions**: For users trading through the Polymarket web app, the relayer pays gas on the user's behalf. For API/SDK users, gas is typically self-paid unless using the relayer (rate limited: 25 requests/minute).

## 5.5 Minimum Order Sizes

Polymarket's CLOB **does not enforce minimum order sizes** by design. The order book matches willing buyers and sellers of any amount. However:
- Liquidity rewards have a "Minimum Shares" threshold below which orders don't earn rewards
- Extremely small orders may be uneconomical due to gas costs (if not using the gasless relayer)

---

# 6. Risks

## 6.1 Resolution Risk

- **Ambiguous resolution criteria**: Markets may have unclear or subjective resolution rules
- **UMA Oracle manipulation**: Large UMA token holders can potentially influence dispute outcomes
- **Cross-platform divergence**: Different platforms may resolve the same event differently (e.g., the 2024 government shutdown case)
- **Mitigation**: Carefully read resolution criteria; avoid markets with ambiguous rules; stick to markets with clear, verifiable outcomes

## 6.2 Smart Contract Risk

- **Bug risk**: Although CTF contracts are audited (Gnosis), no smart contract is provably bug-free
- **Upgrade risk**: Contract upgrades could change behavior
- **Mitigation**: Polymarket uses battle-tested Gnosis CTF contracts; exchange contracts are open-source on GitHub

## 6.3 Regulatory Risk

- **US regulation**: Polymarket received CFTC DCM approval in November 2025, but operates under strict conditions
- **State-level challenges**: Nevada, Tennessee, and Massachusetts have restrictions on certain contract types (sports/gaming adjacent)
- **Insider trading**: Not explicitly criminalized for prediction markets, but high-profile cases have drawn scrutiny
- **Evolving landscape**: Regulatory environment continues to shift; new restrictions could emerge
- **Global**: Some jurisdictions restrict access entirely; geo-blocking is enforced

## 6.4 Liquidity & Slippage Risk

- **Low-liquidity markets**: Wide spreads, high slippage, difficult to enter/exit large positions
- **Finding**: 78% of arbitrage opportunities in low-volume markets fail due to execution inefficiencies
- **Capital lockup**: Funds are locked until market resolution (could be months)
- **Mitigation**: Trade liquid markets; limit position size to fraction of order book depth; use limit orders

## 6.5 Inventory / Directional Risk (Market Makers)

- **Adverse selection**: Informed traders tend to trade against market makers during news events
- **Inventory accumulation**: Persistent directional flow leads to large one-sided positions
- **Binary outcome**: Unlike traditional markets, prediction market positions go to $0 or $1 — total loss is possible
- **Mitigation**: Dynamic spread widening during events; strict inventory limits; quick merging of matched positions

## 6.6 Technical / Operational Risk

- **Heartbeat failure**: If heartbeat is not received within 10 seconds, ALL open orders are cancelled
- **API downtime**: CLOB operator downtime means inability to trade or cancel orders
- **Rate limiting**: Aggressive strategies may hit rate limits during volatile periods
- **Key management**: Private key compromise means total loss of funds
- **Mitigation**: Redundant heartbeat systems; error handling for API failures; secure key storage (hardware wallets, KMS)

## 6.7 Counterparty Risk

- **Minimal**: Smart contract-based settlement eliminates traditional counterparty risk
- **Funds are never held by the platform** — only by transparent code
- **Even if Polymarket shuts down**: On-chain settlement can still execute via the deployed contracts
- **Remaining risk**: Operator could theoretically censor orders or refuse to match (but cannot steal funds)

---

# 7. Fees

## 7.1 Trading Fees

### Fee-Free Markets (Majority of Markets)
- **No fees to trade** — zero maker and taker fees
- **No fees to deposit or withdraw USDC**
- This applies to most Polymarket markets as of March 2026

### Fee-Enabled Markets (Since Early 2026)

**Fee Formula:**
```
fee = C * p * feeRate * (p * (1 - p))^exponent
```
Where: C = shares traded, p = share price

| Market Type | Fee Rate | Exponent | Max Effective Rate (at p=0.50) | Maker Rebate |
|-------------|----------|----------|-------------------------------|--------------|
| **Crypto (all)** | 0.25 | 2 | 1.56% | 20% |
| **Sports (NCAAB, Serie A)** | 0.0175 | 1 | 0.44% | 25% |

**Key Properties:**
- Fee peaks at 50% probability and decreases symmetrically toward 0% and 100%
- Fees on buy orders are collected in shares; fees on sell orders are collected in USDC
- Minimum fee: 0.0001 USDC
- SDK clients automatically calculate fees; REST API users must manually include `feeRateBps` in signed orders

**Rollout Timeline:**
- Jan 19, 2026: 15-min crypto markets
- Feb 12, 2026: 5-min crypto markets
- Feb 18, 2026: NCAAB and Serie A sports markets
- Mar 6, 2026: 1H-weekly crypto markets

### US Regulated Exchange (Polymarket Exchange)
- Taker fee: 10 basis points (0.10%) on Total Contract Premium
- Maker rebate: 10 basis points (0.10%) on Total Contract Premium

## 7.2 Withdrawal Fees

- **Polymarket Global**: 2% of net profits at withdrawal
- **On-chain withdrawal (USDC)**: Only Polygon gas (~$0.01-0.05)
- **Third-party fees**: Coinbase, MoonPay, etc. may charge their own fees for fiat off-ramp

## 7.3 Gas Costs (Polygon)

| Operation | Typical Cost |
|-----------|-------------|
| Standard trade (via relayer) | $0.00 (gasless) |
| Standard trade (direct) | $0.05-0.20 |
| Token approval | $0.01-0.05 |
| Split/Merge/Redeem | $0.05-0.20 |
| Complex transactions | Up to $0.50 |

**Note**: The gasless relayer (used by the web app) has a rate limit of 25 requests/minute. API traders executing at higher frequencies will need to pay gas directly in POL.

---

# 8. Open-Source Tools & Repositories

## 8.1 Official Polymarket Repositories

| Repository | Description |
|------------|-------------|
| [py-clob-client](https://github.com/Polymarket/py-clob-client) | Official Python SDK for CLOB API |
| [@polymarket/clob-client](https://github.com/Polymarket/clob-client) | Official TypeScript SDK for CLOB API |
| [python-order-utils](https://github.com/Polymarket/python-order-utils) | Python utilities for generating/signing orders |
| [ctf-exchange](https://github.com/Polymarket/ctf-exchange) | Exchange smart contract source code |
| [neg-risk-ctf-adapter](https://github.com/Polymarket/neg-risk-ctf-adapter) | NegRisk adapter for multi-outcome markets |
| [agents](https://github.com/Polymarket/agents) | AI agent framework for autonomous Polymarket trading |

## 8.2 Community Trading Bots

| Repository | Description |
|------------|-------------|
| [warproxxx/poly-maker](https://github.com/warproxxx/poly-maker) | Market making bot with Google Sheets config; includes volatility analysis, position management, and merger utility |
| [warproxxx/poly_data](https://github.com/warproxxx/poly_data) | Data retrieval and processing for Polymarket markets, order events, and trades |
| [discountry/polymarket-trading-bot](https://github.com/discountry/polymarket-trading-bot) | Beginner-friendly bot with gasless transactions and WebSocket data |
| [ImMike/polymarket-arbitrage](https://github.com/ImMike/polymarket-arbitrage) | Polymarket-Kalshi arbitrage bot; watches 10,000+ markets for cross-platform inefficiencies |
| [CarlosIbCu/polymarket-kalshi-btc-arbitrage-bot](https://github.com/CarlosIbCu/polymarket-kalshi-btc-arbitrage-bot) | BTC-specific arbitrage bot between Polymarket and Kalshi |
| [speedyhughes/kalshi-poly-arb](https://github.com/speedyhughes/kalshi-poly-arb) | Cross-platform arbitrage (Poly-Poly, Kalshi-Kalshi, Kalshi-Poly) |

## 8.3 Integrations & Frameworks

| Tool | Description |
|------|-------------|
| [NautilusTrader](https://nautilustrader.io/docs/latest/integrations/polymarket/) | Professional algorithmic trading framework with Polymarket integration |
| [polymarket-apis (PyPI)](https://pypi.org/project/polymarket-apis/) | Community Python wrapper for Polymarket APIs |
| [Bitquery Polymarket API](https://docs.bitquery.io/docs/examples/polymarket-api/main-polymarket-contract/) | On-chain data queries for Polymarket contracts |
| [PolymarketAnalytics](https://polymarketanalytics.com) | Analytics dashboards for prediction market data |

## 8.4 Research & Academic Resources

| Resource | Description |
|----------|-------------|
| [arxiv.org/abs/2508.03474](https://arxiv.org/abs/2508.03474) | "Unravelling the Probabilistic Forest: Arbitrage in Prediction Markets" — academic paper documenting $40M+ in arbitrage profits |
| [QuantPedia: Systematic Edges](https://quantpedia.com/systematic-edges-in-prediction-markets/) | Analysis of systematic trading edges in prediction markets |
| [Building a Quantitative Prediction System](https://navnoorbawa.substack.com/p/building-a-quantitative-prediction) | Technical deep-dive on quant systems for Polymarket |

---

# 9. Sources

**Official Documentation:**
- [Polymarket Documentation Hub](https://docs.polymarket.com)
- [CLOB Introduction](https://docs.polymarket.com/developers/CLOB/introduction)
- [CLOB Authentication](https://docs.polymarket.com/developers/CLOB/authentication)
- [WebSocket Overview](https://docs.polymarket.com/developers/CLOB/websocket/wss-overview)
- [API Rate Limits](https://docs.polymarket.com/quickstart/introduction/rate-limits)
- [Trading Fees](https://docs.polymarket.com/polymarket-learn/trading/fees)
- [Conditional Token Framework](https://docs.polymarket.com/trading/ctf/overview)
- [Order Creation](https://docs.polymarket.com/developers/CLOB/orders/create-order)
- [Market Making Trading Guide](https://docs.polymarket.com/developers/market-makers/trading)
- [Maker Rebates Program](https://docs.polymarket.com/developers/market-makers/maker-rebates-program)
- [Liquidity Rewards](https://help.polymarket.com/en/articles/13364466-liquidity-rewards)
- [Builder Program](https://docs.polymarket.com/developers/builders/builder-intro)
- [Endpoints Reference](https://docs.polymarket.com/quickstart/reference/endpoints)
- [Trading Limits](https://docs.polymarket.com/polymarket-learn/trading/no-limits)

**SDKs & Code:**
- [py-clob-client (GitHub)](https://github.com/Polymarket/py-clob-client)
- [py_clob_client Full Reference (AgentBets.ai)](https://agentbets.ai/guides/py-clob-client-reference/)
- [Polymarket GitHub Organization](https://github.com/polymarket)
- [CTF Exchange Contract (GitHub)](https://github.com/Polymarket/ctf-exchange)
- [NegRisk Adapter (GitHub)](https://github.com/Polymarket/neg-risk-ctf-adapter)

**Strategy & Analysis:**
- [Automated Market Making on Polymarket (Polymarket Blog)](https://news.polymarket.com/p/automated-market-making-on-polymarket)
- [Prediction Market Arbitrage Guide 2026](https://newyorkcityservers.com/blog/prediction-market-arbitrage-guide)
- [Arbitrage Opportunities in Prediction Markets (ainvest.com)](https://www.ainvest.com/news/arbitrage-opportunities-prediction-markets-smart-money-profits-price-inefficiencies-polymarket-2512/)
- [Cross-Platform Arbitrage Strategies (AhaSignals)](https://ahasignals.com/research/prediction-market-arbitrage-strategies/)
- [Systematic Edges in Prediction Markets (QuantPedia)](https://quantpedia.com/systematic-edges-in-prediction-markets/)
- [Building a Quantitative Prediction System](https://navnoorbawa.substack.com/p/building-a-quantitative-prediction)
- [Unravelling the Probabilistic Forest (arXiv)](https://arxiv.org/abs/2508.03474)

**Regulatory & Risk:**
- [Polymarket CFTC Approval (CoinDesk)](https://www.coindesk.com/business/2025/11/25/polymarket-secures-cftc-approval-for-regulated-u-s-return/)
- [Polymarket Acquires QCEX (PR Newswire)](https://www.prnewswire.com/news-releases/polymarket-acquires-cftc-licensed-exchange-and-clearinghouse-qcex-for-112-million-302509626.html)
- [Resolution Disputes (Phemex)](https://phemex.com/news/article/polymarket-faces-rule-dispute-over-market-resolution-39945)
- [Polymarket Legal Status (ats.io)](https://ats.io/prediction-markets/polymarket/legal/)

**Fee Analysis:**
- [Polymarket Fees Explained (CryptoNews)](https://cryptonews.com/cryptocurrency/polymarket-fees/)
- [Polymarket Fees 2026 (ballislife.com)](https://ballislife.com/prediction-markets/polymarket/fees/)
- [Understanding the Polymarket Fee Curve (QuantJourney)](https://quantjourney.substack.com/p/understanding-the-polymarket-fee)

**On-Chain Data:**
- [CTF Exchange on PolygonScan](https://polygonscan.com/address/0x4bfb41d5b3570defd03c39a9a4d8de6bd8b8982e)
- [Neg Risk CTF Exchange on PolygonScan](https://polygonscan.com/address/0xc5d563a36ae78145c45a50134d48a1215220f80a)
- [Decoding Polymarket On-Chain Data](https://yzc.me/x01Crypto/decoding-polymarket)
