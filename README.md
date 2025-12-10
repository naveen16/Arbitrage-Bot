# Uniswap Arbitrage Bot - System Design Document

## 1. Executive Summary

This document outlines the design for an off-chain arbitrage bot that monitors Uniswap V2 and V3 pools on Ethereum mainnet, detects profitable price discrepancies, and executes atomic arbitrage trades.

---

## 2. Problem Statement

### What is Arbitrage in DeFi?

Arbitrage exploits price differences for the same asset across different markets. In the context of Uniswap:

- **Same token pair, different pools**: ETH/USDC might trade at $2000 on one V3 pool (0.3% fee) and $2005 on another V3 pool (0.05% fee)
- **Triangular arbitrage**: ETH → USDC → DAI → ETH where the final ETH amount exceeds the input
- **Cross-version arbitrage**: Price differences between V2 and V3 pools for the same pair

### Why is this Hard?

1. **Speed**: Opportunities exist for milliseconds before being captured
2. **Competition**: Sophisticated MEV bots and searchers compete for the same opportunities
3. **Gas costs**: Transaction fees can exceed profit margins
4. **Slippage**: Large trades move prices, reducing expected profit
5. **Failed transactions**: You pay gas even if the trade fails

---

## 3. Requirements Gathering

### Questions I Would Ask the Team

#### Business Requirements
- [ ] **What's our capital budget?** (Affects position sizing and opportunity filtering)
- [ ] **What's our acceptable risk tolerance?** (Do we accept any failed tx gas costs?)
- [ ] **Target ROI/profit per trade?** (Minimum profit threshold after gas)
- [ ] **24/7 operation requirement?** (Uptime SLA)

#### Technical Constraints
- [ ] **Latency budget?** (End-to-end from opportunity detection to tx submission)
- [ ] **Do we have access to private mempools/flashbots?** (Critical for MEV protection)
- [ ] **Existing infrastructure?** (Node access, cloud accounts, monitoring)
- [ ] **On-chain contract deployment allowed?** (For atomic execution)

#### Scope
- [ ] **Which token pairs to monitor?** (Top 100 by liquidity? Specific whitelist?)
- [ ] **V2 only, V3 only, or both?**
- [ ] **Single-hop or multi-hop arbitrage?** (Triangular adds complexity)

---

## 4. Assumptions

For this design, I'm assuming:

| Assumption | Rationale |
|------------|-----------|
| **Capital: $100K-$1M available** | Enough to capture meaningful opportunities, not enough to move markets significantly |
| **Minimum profit: $50 after gas** | Below this, risk/reward isn't worth it |
| **Latency target: <100ms** | Competitive but not HFT-level (would require co-location) |
| **Flashbots/MEV-Share access: Yes** | Essential for protecting against sandwich attacks |
| **Custom smart contract: Yes** | Required for atomic multi-swap execution |
| **Token scope: Top 50 pairs by liquidity** | Balance between opportunity and complexity |
| **Both V2 and V3** | More opportunities, manageable complexity |
| **Two-hop maximum** | Three-hop+ has diminishing returns |

---

## 5. Success Metrics

| Metric | Target | Why It Matters |
|--------|--------|----------------|
| **Opportunity Detection Rate** | >95% of profitable opps | Are we seeing what's out there? |
| **Win Rate** | >80% of submitted txs | Failed txs = wasted gas |
| **Average Profit per Trade** | >$100 | Sustainability |
| **System Uptime** | >99.9% | Opportunities don't wait |
| **End-to-End Latency** | p95 < 50ms | Speed is money |

---

## 6. High-Level Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              ARBITRAGE BOT SYSTEM                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐            │
│  │   Ethereum   │     │   Pool State     │     │   Opportunity    │            │
│  │    Nodes     │────▶│    Indexer       │────▶│    Detector      │            │
│  │ (Geth/Erigon)│     │                  │     │                  │            │
│  └──────────────┘     └──────────────────┘     └────────┬─────────┘            │
│         │                      │                        │                       │
│         │              ┌───────▼───────┐               │                       │
│         │              │  State Cache  │               │                       │
│         │              │   (Redis)     │               │                       │
│         │              └───────────────┘               │                       │
│         │                                               ▼                       │
│         │                                    ┌──────────────────┐              │
│         │                                    │   Profit         │              │
│         │                                    │   Calculator     │              │
│         │                                    │   + Gas Estimator│              │
│         │                                    └────────┬─────────┘              │
│         │                                             │                        │
│         │                                             ▼                        │
│         │    ┌──────────────────┐          ┌──────────────────┐               │
│         │    │  Transaction     │◀─────────│   Execution      │               │
│         │    │  Builder         │          │   Engine         │               │
│         │    └────────┬─────────┘          └──────────────────┘               │
│         │             │                             │                          │
│         │             ▼                             │                          │
│         │    ┌──────────────────┐                  │                          │
│         │    │  MEV Protection  │                  │                          │
│         │    │  (Flashbots)     │                  │                          │
│         │    └────────┬─────────┘                  │                          │
│         │             │                             │                          │
│         ▼             ▼                             ▼                          │
│  ┌─────────────────────────────────────────────────────────────────┐          │
│  │                    ETHEREUM MAINNET                              │          │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │          │
│  │  │ Uniswap V2  │  │ Uniswap V3  │  │ Arbitrage Contract      │ │          │
│  │  │   Pools     │  │   Pools     │  │ (Atomic Execution)      │ │          │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘ │          │
│  └─────────────────────────────────────────────────────────────────┘          │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐          │
│  │                    OBSERVABILITY LAYER                           │          │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │          │
│  │  │  Metrics    │  │   Logs      │  │   Alerting              │ │          │
│  │  │ (Prometheus)│  │ (Loki/ELK) │  │  (PagerDuty)            │ │          │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘ │          │
│  └─────────────────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Core Components

| Component | Responsibility | Why It's Separate |
|-----------|---------------|-------------------|
| **Pool State Indexer** | Subscribes to on-chain events, maintains current reserves/liquidity | Decouples data ingestion from business logic |
| **State Cache (Redis)** | In-memory store for pool states, gas prices, token prices | Sub-millisecond reads; single source of truth |
| **Opportunity Detector** | Compares prices across pools, identifies potential arbs | CPU-intensive; can scale horizontally |
| **Profit Calculator** | Simulates trades, calculates expected profit after gas/slippage | Accuracy is critical; deserves isolation |
| **Execution Engine** | Decides whether to execute, manages position limits | Contains business rules and risk controls |
| **Transaction Builder** | Constructs calldata, sets gas parameters | Complexity warrants separation |
| **MEV Protection** | Routes txs through Flashbots/private pools | Security-critical; pluggable |

### Component Architecture: Microservices vs Monolith

**The components above are logical separations, not necessarily physical services.**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  OPTION A: MICROSERVICES (Multiple Processes)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                    │
│  │  indexer     │   │  detector    │   │  executor    │                    │
│  │  (process 1) │   │  (process 2) │   │  (process 3) │                    │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘                    │
│         │                  │                  │                             │
│         └──────────────────┴──────────────────┘                             │
│                            │                                                │
│                     ┌──────▼──────┐                                        │
│                     │    Redis    │  ← Shared state + pub/sub              │
│                     └─────────────┘                                        │
│                                                                              │
│  Pros: Scale independently, isolated failures, can use different languages │
│  Cons: Network latency (~1ms), operational complexity, more moving parts   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  OPTION B: MONOLITH WITH MODULES (Single Process) ← RECOMMENDED FOR V1     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      arb-bot (single process)                        │   │
│  │                                                                      │   │
│  │   src/                                                               │   │
│  │   ├── indexer/           ← Module (just TypeScript files)           │   │
│  │   │   ├── index.ts                                                  │   │
│  │   │   ├── v2-indexer.ts                                             │   │
│  │   │   └── v3-indexer.ts                                             │   │
│  │   ├── detector/          ← Module                                   │   │
│  │   │   ├── index.ts                                                  │   │
│  │   │   ├── path-finder.ts                                            │   │
│  │   │   └── profit-calc.ts                                            │   │
│  │   ├── executor/          ← Module                                   │   │
│  │   │   ├── index.ts                                                  │   │
│  │   │   ├── tx-builder.ts                                             │   │
│  │   │   └── flashbots.ts                                              │   │
│  │   ├── state/             ← Shared state (in-memory + Redis backup)  │   │
│  │   │   ├── pool-cache.ts                                             │   │
│  │   │   └── redis-client.ts                                           │   │
│  │   └── main.ts            ← Wires everything together                │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                            │                                                │
│                     ┌──────▼──────┐                                        │
│                     │    Redis    │  ← State persistence + recovery        │
│                     └─────────────┘                                        │
│                                                                              │
│  Pros: Zero network latency, simple deployment, easy debugging             │
│  Cons: Single point of failure, can't scale components independently       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Our Recommendation: Option B (Monolith with Modules)**

```typescript
// src/main.ts - How everything wires together

import { PoolIndexer } from './indexer';
import { OpportunityDetector } from './detector';
import { ExecutionEngine } from './executor';
import { StateCache } from './state/pool-cache';
import { RedisClient } from './state/redis-client';

async function main() {
  // Shared dependencies
  const redis = new RedisClient(process.env.REDIS_URL);
  const stateCache = new StateCache(redis);  // In-memory with Redis backup
  
  // Initialize components (they're just classes, not services)
  const indexer = new PoolIndexer(stateCache);
  const detector = new OpportunityDetector(stateCache);
  const executor = new ExecutionEngine();
  
  // Wire them together with callbacks (no network!)
  indexer.onPoolUpdate((pool) => {
    // Direct function call - nanoseconds, not milliseconds
    const opportunities = detector.scan(pool);
    
    for (const opp of opportunities) {
      executor.evaluate(opp);  // Another direct call
    }
  });
  
  // Start the indexer (connects to Ethereum node)
  await indexer.start();
  
  console.log('Arb bot running...');
}

main();
```

**Why Monolith for an Arb Bot?**

| Factor | Microservices | Monolith | Winner for Arb Bot |
|--------|--------------|----------|-------------------|
| Latency | +1-5ms per hop | ~0ms | **Monolith** |
| Debugging | Distributed tracing needed | Single stack trace | **Monolith** |
| Deployment | Multiple containers/services | Single binary | **Monolith** |
| Team size | Good for 10+ engineers | Good for 1-5 engineers | **Monolith** |
| Failure modes | Complex (partial failures) | Simple (all or nothing) | **Monolith** |

**When to Switch to Microservices:**
- Multiple teams working on different components
- Need to scale detector to 10+ instances
- Want different languages (e.g., Rust for hot path)
- Running multiple independent strategies

**Redis Role in Both Architectures:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  REDIS IS NOT THE PRIMARY DATA PATH                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  In our monolith:                                                           │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  StateCache (in-memory Map)         Redis (persistence layer)       │   │
│  │  ────────────────────────           ─────────────────────────       │   │
│  │                                                                      │   │
│  │  • Primary read/write path          • Backup on every update        │   │
│  │  • JavaScript Map<string, Pool>     • Survives process restart      │   │
│  │  • Access time: ~0.001ms            • Access time: ~0.5ms           │   │
│  │                                                                      │   │
│  │  Hot path uses in-memory:           Cold start loads from Redis:    │   │
│  │  stateCache.get('0x...') → Map      redis.hgetall('pools') → Map    │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Write flow:                                                                │
│  1. Indexer receives Sync event                                            │
│  2. Update in-memory Map (instant)                                         │
│  3. Async write to Redis (fire-and-forget, doesn't block hot path)        │
│                                                                              │
│  Read flow:                                                                 │
│  1. Detector reads from in-memory Map                                      │
│  2. Never touches Redis during normal operation                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Component Deep Dives

### 7.1 Pool State Indexer

**Purpose**: Maintain a real-time, accurate view of all monitored Uniswap pool states.

**Why this is tricky**: 
- Ethereum blocks arrive every ~12 seconds, but pool states change *within* blocks
- We need to account for pending transactions because we don't know the execution order
- We need to know state at the *moment of our transaction*, not current block state

**Design Decisions**:

```
┌─────────────────────────────────────────────────────────────┐
│                    POOL STATE INDEXER                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Data Sources (Priority Order):                            │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ 1. WebSocket Subscriptions (newHeads, logs)         │  │
│   │    - Lowest latency for confirmed state             │  │
│   │    - V3: Swap, Mint, Burn events                   │  │
│   │    - V2: Sync events                               │  │
│   └─────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ 2. Pending Transaction Stream (optional)            │  │
│   │    - See upcoming state changes before mining       │  │
│   │    - High noise, but valuable signal                │  │
│   └─────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ 3. Periodic eth_call for reserves (backup)          │  │
│   │    - Fallback if events missed                      │  │
│   │    - Reconciliation check                           │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Where Does Pool Data Come From?**

Pool state comes **directly from Uniswap smart contracts** on Ethereum

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DATA FLOW: Blockchain → Our System                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         ETHEREUM MAINNET                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │   Uniswap V2 Pair Contract              Uniswap V3 Pool Contract    │   │
│  │   ─────────────────────────             ────────────────────────    │   │
│  │   • reserve0: 50,000,000 USDC           • sqrtPriceX96: 1234...     │   │
│  │   • reserve1: 25,000 WETH               • liquidity: 9876...        │   │
│  │                                          • tick: -201234             │   │
│  │                                                                      │   │
│  │   Events emitted on every swap:         Events emitted on swap:     │   │
│  │   └─▶ Sync(reserve0, reserve1)          └─▶ Swap(amount0, amount1,  │   │
│  │                                                sqrtPriceX96, tick)   │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                          │
│                                  │ WebSocket subscription                   │
│                                  │ eth_subscribe("logs", {topics: [...]})  │
│                                  ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        OUR ETHEREUM NODE                            │   │
│  │                        (Erigon / Geth)                              │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                          │
│                                  │ Event stream + RPC calls                │
│                                  ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      POOL STATE INDEXER                             │   │
│  │                                                                      │   │
│  │   On Sync event received:                                           │   │
│  │   1. Parse reserve0, reserve1 from event data                       │   │
│  │   2. Update our local state cache                                   │   │
│  │   3. Notify Opportunity Detector                                    │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**How We Read Pool State (Code Example)**:

```typescript
import { createPublicClient, webSocket } from 'viem';
import { mainnet } from 'viem/chains';

// Connect to our Ethereum node
const client = createPublicClient({
  chain: mainnet,
  transport: webSocket('wss://our-node.example.com'),
});

// V2: Read reserves directly from contract
async function getV2Reserves(pairAddress: string) {
  const [reserve0, reserve1, blockTimestamp] = await client.readContract({
    address: pairAddress,
    abi: UNISWAP_V2_PAIR_ABI,
    functionName: 'getReserves',
  });
  return { reserve0, reserve1, blockTimestamp };
}

// V3: Read slot0 for current price/tick
async function getV3State(poolAddress: string) {
  const [sqrtPriceX96, tick, ...rest] = await client.readContract({
    address: poolAddress,
    abi: UNISWAP_V3_POOL_ABI,
    functionName: 'slot0',
  });
  return { sqrtPriceX96, tick };
}

// Subscribe to V2 Sync events (real-time updates)
client.watchContractEvent({
  address: USDC_WETH_V2_PAIR,
  abi: UNISWAP_V2_PAIR_ABI,
  eventName: 'Sync',
  onLogs: (logs) => {
    for (const log of logs) {
      const { reserve0, reserve1 } = log.args;
      updatePoolState(USDC_WETH_V2_PAIR, reserve0, reserve1);
      triggerOpportunityDetection();
    }
  },
});

// Subscribe to V3 Swap events (real-time updates)
client.watchContractEvent({
  address: USDC_WETH_V3_POOL,
  abi: UNISWAP_V3_POOL_ABI,
  eventName: 'Swap',
  onLogs: (logs) => {
    for (const log of logs) {
      const { sqrtPriceX96, tick } = log.args;
      updateV3PoolState(USDC_WETH_V3_POOL, sqrtPriceX96, tick);
      triggerOpportunityDetection();
    }
  },
});

// Subscribe to V3 Mint events (liquidity added)
client.watchContractEvent({
  address: USDC_WETH_V3_POOL,
  abi: UNISWAP_V3_POOL_ABI,
  eventName: 'Mint',
  onLogs: (logs) => {
    for (const log of logs) {
      const { tickLower, tickUpper, amount } = log.args;
      updateV3Liquidity(USDC_WETH_V3_POOL, tickLower, tickUpper, amount);
      triggerOpportunityDetection();
    }
  },
});

// Subscribe to V3 Burn events (liquidity removed)
client.watchContractEvent({
  address: USDC_WETH_V3_POOL,
  abi: UNISWAP_V3_POOL_ABI,
  eventName: 'Burn',
  onLogs: (logs) => {
    for (const log of logs) {
      const { tickLower, tickUpper, amount } = log.args;
      updateV3Liquidity(USDC_WETH_V3_POOL, tickLower, tickUpper, -amount);
      triggerOpportunityDetection();
    }
  },
});
```

**Key Point**: We're reading the **actual contract state** on Ethereum. This is the source of truth - the exact reserves that our swap would execute against.

**V2 vs V3 Differences**:

| Aspect | Uniswap V2 | Uniswap V3 |
|--------|-----------|-----------|
| **State to track** | `reserve0`, `reserve1` | Liquidity at each tick, current tick, sqrtPriceX96 |
| **Complexity** | Simple constant product | Concentrated liquidity math |
| **Events** | `Sync(reserve0, reserve1)` | `Swap`, `Mint`, `Burn` with tick data |
| **Price calculation** | `reserve1 / reserve0` | Complex tick math |

**Implementation Notes**:
- Use **ethers.js** or **viem** for type-safe contract interactions
- Maintain a **local copy** of pool state; don't call RPC for every price check
- Implement **reorg handling**: if chain reorgs, roll back state to common ancestor

```typescript
// Pseudocode: Pool state structure
interface V2PoolState {
  address: string;
  token0: string;
  token1: string;
  reserve0: bigint;
  reserve1: bigint;
  blockNumber: number;
  lastUpdateTimestamp: number;
}

interface V3PoolState {
  address: string;
  token0: string;
  token1: string;
  sqrtPriceX96: bigint;
  liquidity: bigint;
  tick: number;
  tickSpacing: number;
  fee: number;  // 500, 3000, or 10000 (basis points)
  // Tick bitmap and tick data for accurate simulation
  ticks: Map<number, TickData>;
  blockNumber: number;
}
```

**Example: ETH/USDC Pool State**

```typescript
// Real example: Uniswap V2 ETH/USDC pool
const ethUsdcPool: V2PoolState = {
  address: "0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc",  // Actual V2 pool
  token0: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",  // USDC (lower address)
  token1: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",  // WETH (higher address)
  reserve0: 50_000_000_000000n,      // 50M USDC (6 decimals)
  reserve1: 25_000_000000000000000000n,  // 25,000 ETH (18 decimals)
  blockNumber: 18500000,
  lastUpdateTimestamp: 1699900000,
};
```

**What We Do With This State:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  HOW WE USE V2 POOL STATE                                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. CALCULATE CURRENT PRICE                                             │
│     ─────────────────────────────────────────────────────────────────   │
│     price = reserve0 / reserve1 (adjusted for decimals)                 │
│                                                                          │
│     Example:                                                             │
│     USDC reserves: 50,000,000 USDC                                      │
│     ETH reserves:  25,000 ETH                                           │
│     Price = 50,000,000 / 25,000 = $2,000 per ETH                       │
│                                                                          │
│  2. SIMULATE SWAP OUTPUT (Constant Product Formula: x * y = k)          │
│     ─────────────────────────────────────────────────────────────────   │
│     "If I swap 10 ETH, how much USDC do I get?"                        │
│                                                                          │
│     k = reserve0 × reserve1                                             │
│     k = 50,000,000 × 25,000 = 1,250,000,000,000                        │
│                                                                          │
│     After adding 10 ETH:                                                │
│     new_reserve1 = 25,000 + 10 = 25,010 ETH                            │
│     new_reserve0 = k / new_reserve1 = 49,980,007.99 USDC               │
│                                                                          │
│     USDC out = 50,000,000 - 49,980,007.99 = 19,992 USDC                │
│     (minus 0.3% fee = 19,932 USDC actual)                              │
│                                                                          │
│     Effective price: $1,993.20 per ETH (worse than spot due to impact) │
│                                                                          │
│  3. COMPARE ACROSS POOLS                                                │
│     ─────────────────────────────────────────────────────────────────   │
│     Pool A (V2):     ETH price = $2,000                                │
│     Pool B (V3 0.3%): ETH price = $2,015                               │
│                                                                          │
│     Arbitrage: Buy ETH on Pool A, sell on Pool B                       │
│     Gross profit: $15 per ETH traded                                   │
│                                                                          │
│  4. DETECT STATE CHANGES                                                │
│     ─────────────────────────────────────────────────────────────────   │
│     When blockNumber changes → reserves changed → recalculate prices   │
│     Compare old vs new state to trigger opportunity detection          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Step-by-Step Arbitrage Execution: $10,000 USDC Example**

Let's walk through exactly what happens when we execute this arbitrage with $10K:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ARBITRAGE EXECUTION FLOW: $10,000 USDC INPUT                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  STARTING STATE:                                                            │
│  ───────────────                                                            │
│  • We have: $10,000 USDC in our wallet                                     │
│  • Pool A (V2):  ETH = $2,000  |  Reserves: 50M USDC / 25K ETH             │
│  • Pool B (V3):  ETH = $2,015  |  Reserves: 30M USDC / 14.9K ETH           │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                              │
│  STEP 1: BUY ETH ON POOL A (cheaper pool)                                  │
│  ────────────────────────────────────────                                   │
│                                                                              │
│    Input:  10,000 USDC                                                      │
│                                                                              │
│    Calculation (V2 constant product with 0.3% fee):                        │
│    • amountInWithFee = 10,000 × 0.997 = 9,970 USDC (after fee)            │
│    • k = 50,000,000 × 25,000 = 1,250,000,000,000                           │
│    • new USDC reserve = 50,000,000 + 10,000 = 50,010,000                   │
│    • new ETH reserve = k / 50,010,000 = 24,995.001 ETH                     │
│    • ETH received = 25,000 - 24,995.001 = 4.999 ETH                        │
│                                                                              │
│    Account for fees:                                                       │
│    • Formula: amountOut = (amountInWithFee × reserveOut) /                │
│               (reserveIn + amountInWithFee)                                 │
│    • ETH out = (9,970 × 25,000) / (50,000,000 + 9,970)                     │
│    • ETH out = 249,250,000 / 50,009,970 = 4.9835 ETH                       │
│                                                                              │
│    Output: 4.9835 ETH                                                       │
│    Effective price: $10,000 / 4.9835 = $2,006.62 per ETH                   │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                              │
│  STEP 2: SELL ETH ON POOL B (expensive pool)                               │
│  ───────────────────────────────────────────                                │
│                                                                              │
│    Input:  4.9835 ETH (everything we just bought)                          │
│                                                                              │
│    Calculation (V3 with 0.3% fee tier):                                    │
│    • For V3, assume concentrated liquidity gives better execution          │
│    • amountInWithFee = 4.9835 × 0.997 = 4.9686 ETH                         │
│    • At $2,015 spot price with minimal slippage (deep liquidity):         │
│    • USDC out ≈ 4.9686 × $2,012 = $9,996.79                               │
│    • (Slight price impact, effective rate ~$2,012 not $2,015)             │
│                                                                              │
│    Output: $9,996.79 USDC                                                   │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                              │
│  WAIT - THAT'S A LOSS! Let's recalculate with realistic numbers...        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Reality Check: Why $15 spread ≠ $15 profit per ETH**

The naive calculation suggests $15/ETH × 5 ETH = $75 profit. But:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  REALISTIC PROFIT CALCULATION                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  COSTS THAT EAT INTO PROFIT:                                               │
│  ──────────────────────────                                                 │
│                                                                              │
│  1. Pool A Fee (0.3%):        $10,000 × 0.003    = -$30.00                 │
│  2. Pool B Fee (0.3%):        $10,033 × 0.003    = -$30.10                 │
│  3. Price Impact Pool A:      ~0.02% on $10K     = -$2.00                  │
│  4. Price Impact Pool B:      ~0.02% on $10K     = -$2.00                  │
│  5. Gas Cost:                 ~150K gas × 50 gwei × $2000/ETH = -$15.00   │
│  6. Priority Fee (Flashbots): ~0.01 ETH tip      = -$20.00                 │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  REVISED CALCULATION (with $15 price difference = 0.75%):                  │
│                                                                              │
│  Gross arbitrage value:       $10,000 × 0.75%    = +$75.00                 │
│  Pool A fee:                                       = -$30.00                │
│  Pool B fee:                                       = -$30.00                │
│  Combined price impact:                            = -$4.00                 │
│  Gas + Priority:                                   = -$35.00                │
│  ───────────────────────────────────────────────────────────────────────   │
│  NET PROFIT (before slippage):                     = -$24.00  ❌ LOSS!     │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                              │
│  THIS IS WHY MOST "OBVIOUS" ARBITRAGE DOESN'T WORK!                        │
│                                                                              │
│  For $10K to be profitable, we need:                                       │
│  • Price difference > 0.6% (fees) + gas/size                               │
│  • Minimum spread needed: ~0.95% ($19 per ETH, not $15)                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**When DOES $10K Arbitrage Work?**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PROFITABLE SCENARIO: $50 price difference (2.5% spread)                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Pool A: ETH = $2,000                                                       │
│  Pool B: ETH = $2,050  (2.5% higher)                                       │
│                                                                              │
│  STEP 1: Buy 4.985 ETH on Pool A for $10,000                               │
│          (after 0.3% fee, effective price $2,006)                          │
│                                                                              │
│  STEP 2: Sell 4.985 ETH on Pool B                                          │
│          Gross: 4.985 × $2,050 = $10,219.25                                │
│          After 0.3% fee: $10,188.59                                        │
│                                                                              │
│  GROSS PROFIT:  $10,188.59 - $10,000 = $188.59                             │
│                                                                              │
│  SUBTRACT COSTS:                                                            │
│  • Gas (150K × 30 gwei × $2000): -$9.00                                    │
│  • Flashbots tip (0.005 ETH):    -$10.00                                   │
│  • Price impact (~0.04%):        -$4.00                                    │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│  NET PROFIT:                      $165.59  ✅                               │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  ROI: 1.66% on $10K in one atomic transaction (~12 seconds)                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Insight**: The relationship between trade size and minimum spread is a **U-shaped curve**, not linear!

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  WHY IT'S A U-SHAPED CURVE (not "bigger = better")                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Min Spread                                                                  │
│  Needed                                                                      │
│     │                                                                        │
│  4% │ ●                                                                      │
│     │  ╲                                                                     │
│  3% │   ╲                                                                    │
│     │    ╲                                                                   │
│  2% │     ╲                                                                  │
│     │      ╲                                                                 │
│  1% │       ╲____●_____                                                     │
│     │              ╲    ╲                                                    │
│ .6% │               ●    ╲                                                  │
│     │                     ●────●                                            │
│     └───────────────────────────────────────────────────────────────────    │
│         $1K    $10K   $100K   $500K   $1M+                                  │
│                                                                              │
│         ◄─── GAS ───►◄─── SWEET ──►◄─── PRICE IMPACT ───►                 │
│         DOMINATES     SPOT         DOMINATES                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Two Competing Forces:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  FORCE 1: FIXED COSTS (Gas) - Favors LARGER trades                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Gas cost is FIXED regardless of trade size (~$15-50)                       │
│                                                                              │
│  Trade Size   │  Gas Cost  │  Gas as % of Trade                            │
│  ──────────────────────────────────────────────────────────────────────    │
│  $1,000       │  $25       │  2.5%  ← Gas alone eats most profit!          │
│  $10,000      │  $25       │  0.25%                                         │
│  $100,000     │  $25       │  0.025% ← Gas becomes negligible              │
│                                                                              │
│  ➜ For fixed costs alone, BIGGER IS BETTER                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  FORCE 2: PRICE IMPACT (Slippage) - Favors SMALLER trades                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Price impact GROWS with trade size (you're moving the market!)            │
│                                                                              │
│  For a pool with $50M liquidity:                                           │
│                                                                              │
│  Trade Size   │  Price Impact  │  Why                                      │
│  ──────────────────────────────────────────────────────────────────────    │
│  $1,000       │  0.002%        │  Tiny ripple                              │
│  $10,000      │  0.02%         │  Small splash                             │
│  $100,000     │  0.2%          │  Noticeable wave                          │
│  $1,000,000   │  2.0%          │  You ARE the market!                      │
│                                                                              │
│  ➜ Your own trade erases the arbitrage opportunity as you execute it!     │
│                                                                              │
│  Example at $1M trade:                                                      │
│  • You see: 2% price difference between pools                              │
│  • You trade: Your buy moves Pool A price UP 1%                            │
│  • You trade: Your sell moves Pool B price DOWN 1%                         │
│  • Result: The 2% gap closed to 0% before you finished!                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**The Optimal Trade Size ("Sweet Spot")**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  FINDING THE SWEET SPOT                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Total Cost = Fixed Costs + Variable Costs (Price Impact)                  │
│                                                                              │
│  Cost %                                                                      │
│     │                                                                        │
│  3% │╲                      ╱                                               │
│     │ ╲    Total Cost     ╱                                                 │
│  2% │  ╲      Curve      ╱                                                  │
│     │   ╲              ╱                                                    │
│  1% │    ╲____╲  ╱____╱    ← OPTIMAL ZONE                                  │
│     │         ╲╱                                                            │
│ .5% │          ●  ← Minimum cost point                                     │
│     │                                                                        │
│     └───────────────────────────────────────────────────────────────────    │
│         $1K       $50K      $500K     $2M                                   │
│                     │                                                        │
│              Optimal trade size                                             │
│              (depends on pool liquidity)                                    │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  PRACTICAL TABLE:                                                           │
│                                                                              │
│  Trade Size    │  Min Spread  │  Dominant Cost    │  Notes                 │
│  ──────────────────────────────────────────────────────────────────────    │
│  $1,000        │  ~3.5%       │  Gas (fixed)      │  Rarely profitable     │
│  $10,000       │  ~1.0%       │  Balanced         │  Common sweet spot     │
│  $100,000      │  ~0.65%      │  Balanced         │  Best efficiency       │
│  $500,000      │  ~0.7%       │  Price impact     │  Starting to hurt      │
│  $1,000,000    │  ~1.0%+      │  Price impact     │  Need deep liquidity   │
│  $5,000,000+   │  ~2.0%+      │  Price impact     │  Only for whale pools  │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  This is why our PROFIT CALCULATOR finds OPTIMAL trade size for each       │
│  opportunity - not just "profitable or not" but "what's the best amount?"  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```typescript
// Helper functions we'd implement
function getSpotPrice(pool: V2PoolState): number {
  // Adjust for decimals: USDC has 6, ETH has 18
  const reserve0Adjusted = Number(pool.reserve0) / 1e6;   // USDC
  const reserve1Adjusted = Number(pool.reserve1) / 1e18;  // ETH
  return reserve0Adjusted / reserve1Adjusted;  // Price in USDC per ETH
}

function getAmountOut(
  amountIn: bigint, 
  reserveIn: bigint, 
  reserveOut: bigint
): bigint {
  // Uniswap V2 formula with 0.3% fee
  const amountInWithFee = amountIn * 997n;  // 0.3% fee
  const numerator = amountInWithFee * reserveOut;
  const denominator = reserveIn * 1000n + amountInWithFee;
  return numerator / denominator;
}

// Usage: "How much USDC for 10 ETH?"
const usdcOut = getAmountOut(
  10n * 10n**18n,           // 10 ETH in wei
  ethUsdcPool.reserve1,     // ETH reserve
  ethUsdcPool.reserve0      // USDC reserve
);
// Result: ~19,932 USDC (accounting for 0.3% fee + price impact)
```

---

### 7.2 Opportunity Detector

**Purpose**: Continuously scan pool states to find profitable arbitrage paths.

**Key Insight**: This is essentially a graph problem where:
- **Nodes** = Tokens
- **Edges** = Pools (with exchange rates as weights)
- **Arbitrage** = Finding cycles where the product of exchange rates > 1

**Arbitrage Types We Support**:

```
1. DIRECT ARBITRAGE (Same pair, different pools)
   
   Pool A (V2)          Pool B (V3, 0.3%)
   ┌─────────┐          ┌─────────┐
   │ ETH/USDC│          │ ETH/USDC│
   │ $2000   │          │ $2010   │
   └─────────┘          └─────────┘
        │                    │
        └──── Buy low ───────┘
             Sell high
   
2. TRIANGULAR ARBITRAGE
   
         USDC
        /    \
    Pool A    Pool B
      /        \
    ETH ─────── DAI
        Pool C
   
   Path: ETH → USDC → DAI → ETH
   Profit if: (rate_A × rate_B × rate_C) > 1
```

**Algorithm Choice**:

| Algorithm | Pros | Cons | Our Choice |
|-----------|------|------|------------|
| Brute Force Cycles | Simple, exhaustive | Exponential in cycle length | ✅ for 2-hop |
| Priority Queue | Only evaluates promising paths | More complex | ✅ for 3+ hop |

**Decision**: For 2-hop max (our assumption), brute force is fast enough. With 50 pairs and ~200 pools, we're looking at O(200²) = 40,000 comparisons per scan. At 1μs per comparison, that's 40ms - acceptable.

**Optimization - Incremental Updates**:
Instead of re-scanning all pairs on every block, only re-evaluate paths involving pools that changed.

```typescript
// Type definitions for arbitrage paths
type TokenAddress = string; // e.g., "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2" (WETH)
type PoolAddress = string; // e.g., "0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640" (ETH/USDC V3 0.05%)

interface Swap {
  poolAddress: PoolAddress;
  poolType: 'V2' | 'V3';
  tokenIn: TokenAddress;
  tokenOut: TokenAddress;
  fee?: number; // For V3: 500, 3000, or 10000 (basis points)
}

interface ArbitragePath {
  swaps: Swap[]; // Array of swaps in sequence
  inputToken: TokenAddress; // Starting token
  outputToken: TokenAddress; // Ending token (should be same as input for arbitrage)
  expectedProfit: number; // Calculated profit (starts at 0, calculated later)
}

type AffectedPaths = ArbitragePath[]; // Array of paths that involve the updated pool's tokens

// Pseudocode: Incremental opportunity detection
class OpportunityDetector {
  private poolGraph: Map<string, Pool[]>;  // token -> pools containing that token
  
  onPoolUpdate(pool: Pool) {
    // Only check paths involving this pool's tokens
    const affectedPaths: AffectedPaths = this.getPathsInvolving(pool.token0, pool.token1);
    
    for (const path of affectedPaths) {
      const profit = this.calculatePathProfit(path);
      if (profit > MIN_PROFIT_THRESHOLD) {
        this.emitOpportunity(path, profit);
      }
    }
  }
  
  getPathsInvolving(token0: TokenAddress, token1: TokenAddress): AffectedPaths {
    // Returns all arbitrage paths that involve token0 or token1
    // Examples:
    // - Direct: ETH/USDC Pool A → ETH/USDC Pool B
    // - Triangular: ETH → USDC → DAI → ETH
    // - Cross-tier: ETH/USDC V3 0.05% → ETH/USDC V3 0.3%
    // ... implementation details
  }
}
```

**Example `affectedPaths` when ETH/USDC pool updates:**

```typescript
[
  // Direct arbitrage: ETH/USDC Pool A vs Pool B
  {
    swaps: [
      {
        poolAddress: "0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640", // ETH/USDC V3 0.05%
        poolType: "V3",
        tokenIn: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2", // WETH
        tokenOut: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", // USDC
        fee: 500
      },
      {
        poolAddress: "0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D8", // ETH/USDC V3 0.3%
        poolType: "V3",
        tokenIn: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", // USDC
        tokenOut: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2", // WETH
        fee: 3000
      }
    ],
    inputToken: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2", // WETH
    outputToken: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2", // WETH (back to start)
    expectedProfit: 0 // Will be calculated
  },
  // ... more paths (triangular arbitrage, etc.)
]
```

---

### 7.3 Profit Calculator

**Purpose**: Given a potential arbitrage path, calculate the **exact** expected profit after all costs.

Getting this wrong means:
- Overestimate → Execute unprofitable trades → Lose money
- Underestimate → Miss profitable opportunities → Leave money on table

**Components of Profit Calculation**:

```
┌────────────────────────────────────────────────────────────────┐
│                    PROFIT CALCULATION                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Gross Profit = Output Amount - Input Amount                   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ COSTS TO SUBTRACT:                                       │  │
│  │                                                          │  │
│  │  1. Gas Cost                                             │  │
│  │     = gasUsed × gasPrice × ETH_PRICE                    │  │
│  │     - Estimate via eth_estimateGas or simulation        │  │
│  │     - Add buffer (1.1x) for safety                      │  │
│  │                                                          │  │
│  │  2. DEX Fees                                             │  │
│  │     - V2: 0.3% per swap                                 │  │
│  │     - V3: 0.01%, 0.05%, 0.3%, or 1% depending on pool   │  │
│  │                                                          │  │
│  │  3. Slippage (for large trades)                         │  │
│  │     - Price impact of our own trade                     │  │
│  │     - Must simulate against current reserves            │  │
│  │                                                          │  │
│  │  4. Priority Fee (for faster inclusion)                 │  │
│  │     - Competitive bidding in MEV environment            │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Net Profit = Gross Profit - Gas - Fees - Slippage - Priority │
│                                                                 │
│  EXECUTE ONLY IF: Net Profit > MIN_THRESHOLD ($50)            │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

**Optimal Trade Size Calculation**:

For a given opportunity, there's an **optimal input amount** that maximizes profit. Too small = fees dominate. Too large = slippage eats profits.

```
Profit vs Trade Size (conceptual)
    │
  P │        ╭───╮
  r │       ╱     ╲
  o │      ╱       ╲
  f │     ╱         ╲
  i │────╱           ╲────── Break-even
  t │   ╱             ╲
    │  ╱               ╲
    └─────────────────────── Trade Size
               ↑
         Optimal size
```

**Implementation**: Use binary search or calculus-based optimization to find optimal input.

```typescript
function findOptimalTradeSize(
  path: ArbPath,
  maxInput: bigint,
  gasPrice: bigint
): { optimalInput: bigint; expectedProfit: bigint } {
  
  let left = MIN_TRADE_SIZE;
  let right = maxInput;
  let bestProfit = 0n;
  let bestInput = 0n;
  
  // Binary search for optimal
  while (right - left > PRECISION) {
    const mid = (left + right) / 2n;
    const profit = simulateTrade(path, mid, gasPrice);
    
    if (profit > bestProfit) {
      bestProfit = profit;
      bestInput = mid;
    }
    
    // Check if we should search higher or lower
    const profitHigher = simulateTrade(path, mid + STEP, gasPrice);
    if (profitHigher > profit) {
      left = mid;
    } else {
      right = mid;
    }
  }
  
  return { optimalInput: bestInput, expectedProfit: bestProfit };
}
```

---

### 7.4 Execution Engine

**Purpose**: Orchestrate the decision to execute and manage risk.

**Key Responsibilities**:
1. **Go/No-Go Decision**: Final gate before execution
2. **Position Limits**: Don't over-expose to any single token
3. **Rate Limiting**: Don't spam transactions
4. **Nonce Management**: Critical for transaction ordering

**State Machine**:

```
                    ┌─────────────┐
                    │   IDLE      │
                    └──────┬──────┘
                           │ Opportunity Received
                           ▼
                    ┌─────────────┐
                    │  VALIDATING │──── Invalid ────▶ IDLE
                    └──────┬──────┘
                           │ Valid
                           ▼
                    ┌─────────────┐
                    │  SIMULATING │──── Unprofitable ──▶ IDLE
                    └──────┬──────┘
                           │ Profitable
                           ▼
                    ┌─────────────┐
                    │  EXECUTING  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ SUCCESS  │ │ REVERTED │ │ TIMEOUT  │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
             └────────────┴────────────┘
                          │
                          ▼
                    ┌─────────────┐
                    │    IDLE     │
                    └─────────────┘
```

**Nonce Management** (Critical!):

```typescript
class NonceManager {
  private currentNonce: number;
  private pendingTxs: Map<number, Transaction>;
  
  async getNextNonce(): Promise<number> {
    // Start from on-chain nonce
    const onChainNonce = await provider.getTransactionCount(wallet, 'pending');
    
    // Account for pending transactions we've sent
    this.currentNonce = Math.max(this.currentNonce, onChainNonce);
    
    return this.currentNonce++;
  }
  
  // Handle stuck transactions
  async replaceStuckTx(nonce: number, newGasPrice: bigint) {
    // Send replacement tx with same nonce but higher gas
  }
}
```

---

### 7.5 Transaction Builder

**Purpose**: Construct the actual transaction calldata for atomic execution.

**Why Atomic Execution Matters**:
Without atomicity, a multi-step arbitrage could fail halfway:
1. Buy ETH with USDC ✓
2. Sell ETH for DAI ✗ (price moved)
3. Now we're stuck holding ETH we didn't want

**Solution: Custom Smart Contract**

```solidity
// Simplified Arbitrage Contract
contract ArbitrageExecutor {
    address public owner;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    // Execute arbitrary swap sequence atomically
    function executeArbitrage(
        address[] calldata targets,
        bytes[] calldata calldatas,
        uint256 minProfit
    ) external onlyOwner {
        uint256 startBalance = address(this).balance;
        
        // Execute all swaps
        for (uint i = 0; i < targets.length; i++) {
            (bool success, ) = targets[i].call(calldatas[i]);
            require(success, "Swap failed");
        }
        
        // Verify profit (or entire tx reverts)
        uint256 endBalance = address(this).balance;
        require(endBalance >= startBalance + minProfit, "Insufficient profit");
    }
    
    // Withdraw profits
    function withdraw() external onlyOwner {
        payable(owner).transfer(address(this).balance);
    }
}
```

**Transaction Construction**:

```typescript
function buildArbitrageTx(
  path: ArbPath,
  inputAmount: bigint,
  minProfit: bigint
): Transaction {
  
  const swapCalls = path.swaps.map(swap => {
    if (swap.poolType === 'V2') {
      return encodeV2Swap(swap);
    } else {
      return encodeV3Swap(swap);
    }
  });
  
  const calldata = arbitrageContract.interface.encodeFunctionData(
    'executeArbitrage',
    [
      swapCalls.map(s => s.target),
      swapCalls.map(s => s.data),
      minProfit
    ]
  );
  
  return {
    to: ARBITRAGE_CONTRACT_ADDRESS,
    data: calldata,
    gasLimit: estimatedGas * 110n / 100n,  // 10% buffer
    maxFeePerGas: currentBaseFee + priorityFee,
    maxPriorityFeePerGas: priorityFee,
  };
}
```

---

### 7.6 MEV Protection Layer

**The Problem**: If we submit a transaction to the public mempool, MEV bots can:
1. **Frontrun**: Execute the same arb before us
2. **Sandwich**: Manipulate price before and after our tx
3. **Copy**: Replicate our strategy

**Solution: Flashbots Protect / Private Mempools**

```
┌─────────────────────────────────────────────────────────────┐
│                   MEV PROTECTION FLOW                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Traditional (DANGEROUS):                                   │
│  ┌────────┐    ┌──────────┐    ┌─────────┐                │
│  │ Our Tx │───▶│ Mempool  │───▶│ Mined   │                │
│  └────────┘    └──────────┘    └─────────┘                │
│                     │                                       │
│                     ▼ (visible to everyone)                │
│               ┌──────────┐                                  │
│               │ MEV Bots │ (can frontrun/sandwich)         │
│               └──────────┘                                  │
│                                                              │
│  With Flashbots:                                            │
│  ┌────────┐    ┌──────────────┐    ┌─────────────┐        │
│  │ Our Tx │───▶│ Flashbots    │───▶│ Block       │        │
│  └────────┘    │ Relay        │    │ Builder     │        │
│                └──────────────┘    └──────┬──────┘        │
│                                           │                 │
│                                           ▼                 │
│                                    ┌─────────────┐         │
│                                    │ Proposer    │         │
│                                    │ (Validator) │         │
│                                    └─────────────┘         │
│                                                              │
│  Benefits:                                                  │
│  ✓ Tx not visible in mempool                               │
│  ✓ No gas cost if tx would revert                          │
│  ✓ Can specify min block for inclusion                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Implementation**:

```typescript
import { Flashbots } from '@flashbots/ethers-provider-bundle';

async function submitViaFlashbots(tx: Transaction) {
  const flashbotsProvider = await Flashbots.create(
    provider,
    wallet,
    FLASHBOTS_RELAY_URL
  );
  
  const signedBundle = await flashbotsProvider.signBundle([
    { signer: wallet, transaction: tx }
  ]);
  
  const targetBlock = await provider.getBlockNumber() + 1;
  
  const simulation = await flashbotsProvider.simulate(
    signedBundle,
    targetBlock
  );
  
  if (simulation.firstRevert) {
    console.log('Simulation reverted, not submitting');
    return;
  }
  
  // Submit to multiple consecutive blocks for better inclusion
  for (let block = targetBlock; block < targetBlock + 3; block++) {
    await flashbotsProvider.sendRawBundle(signedBundle, block);
  }
}
```

---

## 8. Data Flow

### End-to-End Flow Diagram

```
TIME ──────────────────────────────────────────────────────────────────▶

     Block N Mined
          │
          ▼
┌─────────────────┐
│ 1. Pool Indexer │  ← WebSocket: new block header
│    receives     │  ← Fetch logs for Swap/Sync events
│    block event  │
└────────┬────────┘
         │ ~10ms
         ▼
┌─────────────────┐
│ 2. Parse events │  ← Decode event data
│    Update Redis │  ← Update pool states atomically
│                 │
└────────┬────────┘
         │ ~5ms
         ▼
┌─────────────────┐
│ 3. Opportunity  │  ← Scan affected token pairs
│    Detector     │  ← Calculate potential profits
│    triggered    │
└────────┬────────┘
         │ ~15ms
         ▼
┌─────────────────┐
│ 4. Profit Calc  │  ← Simulate exact output amounts
│    validates    │  ← Calculate gas costs
│    opportunity  │  ← Find optimal trade size
└────────┬────────┘
         │ ~10ms
         ▼
┌─────────────────┐
│ 5. Execution    │  ← Check risk limits
│    Engine       │  ← Final go/no-go
│    decides      │
└────────┬────────┘
         │ ~5ms
         ▼
┌─────────────────┐
│ 6. Tx Builder   │  ← Construct calldata
│    creates tx   │  ← Set gas parameters
│                 │
└────────┬────────┘
         │ ~5ms
         ▼
┌─────────────────┐
│ 7. Flashbots    │  ← Sign bundle
│    submission   │  ← Submit to relay
│                 │
└────────┬────────┘
         │
         ▼
    Total: ~50ms from block to submission
         │
         │  ... waiting for inclusion ...
         │
         ▼
     Block N+1 or N+2: TX MINED
```

---

## 9. Cloud Infrastructure & Deployment

### Design Philosophy

For an arbitrage bot, infrastructure decisions are driven by one principle: **latency is money**. Every millisecond between opportunity detection and transaction submission is a chance for someone else to capture it first.

### Infrastructure Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CLOUD INFRASTRUCTURE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  REGION: us-east-1 (AWS) or equivalent                                      │
│  WHY: Closest to major Ethereum node providers and Flashbots relays         │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         VPC / Private Network                        │   │
│  │                                                                      │   │
│  │  ┌──────────────────┐      ┌──────────────────┐                    │   │
│  │  │   PRIMARY NODE   │      │   BACKUP NODE    │                    │   │
│  │  │   (Dedicated)    │      │   (Alchemy/Infura)│                   │   │
│  │  │                  │      │                  │                    │   │
│  │  │  Erigon Archive  │      │  WebSocket       │                    │   │
│  │  │  + Lighthouse    │      │  Fallback        │                    │   │
│  │  └────────┬─────────┘      └────────┬─────────┘                    │   │
│  │           │                          │                              │   │
│  │           └──────────┬───────────────┘                              │   │
│  │                      ▼                                              │   │
│  │  ┌─────────────────────────────────────────────────────────────┐  │   │
│  │  │                    COMPUTE TIER                              │  │   │
│  │  │                                                              │  │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │  │   │
│  │  │  │ Pool        │  │ Opportunity │  │ Execution           │ │  │   │
│  │  │  │ Indexer     │  │ Detector    │  │ Engine              │ │  │   │
│  │  │  │             │  │             │  │                     │ │  │   │
│  │  │  │ c6i.xlarge  │  │ c6i.2xlarge │  │ c6i.xlarge          │ │  │   │
│  │  │  │ (4 vCPU)    │  │ (8 vCPU)    │  │ (4 vCPU)            │ │  │   │
│  │  │  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │  │   │
│  │  │         │                │                     │            │  │   │
│  │  │         └────────────────┼─────────────────────┘            │  │   │
│  │  │                          ▼                                  │  │   │
│  │  │              ┌─────────────────────┐                        │  │   │
│  │  │              │   Redis Cluster     │                        │  │   │
│  │  │              │   (ElastiCache)     │                        │  │   │
│  │  │              │   r6g.large         │                        │  │   │
│  │  │              └─────────────────────┘                        │  │   │
│  │  │                                                              │  │   │
│  │  └─────────────────────────────────────────────────────────────┘  │   │
│  │                                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────┐  │   │
│  │  │                   OBSERVABILITY                              │  │   │
│  │  │                                                              │  │   │
│  │  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐ │  │   │
│  │  │  │Prometheus │  │  Grafana  │  │   Loki    │  │PagerDuty │ │  │   │
│  │  │  │ (metrics) │  │ (dashbrd) │  │  (logs)   │  │ (alerts) │ │  │   │
│  │  │  └───────────┘  └───────────┘  └───────────┘  └──────────┘ │  │   │
│  │  │                                                              │  │   │
│  │  └─────────────────────────────────────────────────────────────┘  │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     EXTERNAL CONNECTIONS                            │   │
│  │                                                                      │   │
│  │  ┌─────────────────────────────┐  ┌─────────────────────────────┐  │   │
│  │  │  Flashbots Relay            │  │  Ethereum Mainnet           │  │   │
│  │  │                             │  │  (via our node)             │  │   │
│  │  │  Core: Private tx submission│  │                             │  │   │
│  │  │  Optional: MEV-Share        │  │  • Pool state (reserves)    │  │   │
│  │  │                             │  │  • Transaction execution    │  │   │
│  │  │  • Submit private txs       │  │  • Event subscriptions      │  │   │
│  │  │  • Bundle submissions       │  │                             │  │   │
│  │  │  • MEV protection           │  │                             │  │   │
│  │  └─────────────────────────────┘  └─────────────────────────────┘  │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Infrastructure Decisions

#### Decision 1: Dedicated Node vs. RPC Provider

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ETHEREUM NODE STRATEGY                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  OPTION A: Dedicated Node (Erigon)                                         │
│  ─────────────────────────────────                                          │
│  Pros:                                                                       │
│  ✓ Lowest latency (~5ms to query, vs ~8-10ms for Geth)                     │
│  ✓ Faster query performance (optimized storage, better for read-heavy)     │
│  ✓ No rate limits                                                           │
│  ✓ Full archive data access                                                 │
│  ✓ Can run custom tracing/simulation                                       │
│  ✓ Chosen for speed: 3-5ms latency savings critical for arbitrage          │
│                                                                              │
│  Cons:                                                                       │
│  ✗ Expensive: 2TB NVMe SSD + 32GB RAM + maintenance                        │
│  ✗ Sync time: Days to weeks                                                │
│  ✗ Operational burden                                                       │
│                                                                              │
│  Cost: ~$800-1500/month (bare metal or high-spec cloud)                    │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  OPTION B: Premium RPC (Alchemy/QuickNode)                                 │
│  ─────────────────────────────────────────                                  │
│  Pros:                                                                       │
│  ✓ No maintenance                                                           │
│  ✓ Instant setup                                                            │
│  ✓ Global distribution                                                      │
│                                                                              │
│  Cons:                                                                       │
│  ✗ Higher latency (~20-50ms)                                               │
│  ✗ Rate limits (even on paid plans)                                        │
│  ✗ Shared infrastructure                                                    │
│                                                                              │
│  Cost: ~$200-500/month for high-throughput plan                            │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  OUR CHOICE: HYBRID                                                         │
│  ─────────────────                                                          │
│  • Primary: Dedicated Erigon node (for speed)                              │
│  • Backup: Alchemy WebSocket (for redundancy)                              │
│  • Rationale: Latency is profit; the extra $500/mo pays for itself         │
│               if we capture even one extra opportunity per week            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Decision 2: Instance Sizing & Why

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  COMPUTE SIZING RATIONALE                                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Component        │ Instance    │ Specs          │ Why This Size?          │
│  ──────────────────────────────────────────────────────────────────────    │
│  Pool Indexer     │ c6i.xlarge  │ 4 vCPU, 8GB    │ I/O bound (WebSocket)   │
│                   │             │                │ Needs fast event loop   │
│  ──────────────────────────────────────────────────────────────────────    │
│  Opportunity      │ c6i.2xlarge │ 8 vCPU, 16GB   │ CPU bound (math)        │
│  Detector         │             │                │ Parallel path scanning  │
│  ──────────────────────────────────────────────────────────────────────    │
│  Execution Engine │ c6i.xlarge  │ 4 vCPU, 8GB    │ Light compute, but      │
│                   │             │                │ needs fast networking   │
│  ──────────────────────────────────────────────────────────────────────    │
│  Redis            │ r6g.large   │ 2 vCPU, 16GB   │ Memory for 200+ pools   │
│  (ElastiCache)    │             │                │ Sub-ms reads critical   │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  WHY c6i (Intel) OVER GRAVITON (ARM)?                                      │
│  • Better single-thread performance                                         │
│  • Our code is latency-sensitive, not throughput-sensitive                 │
│  • Crypto libraries (signing) optimized for x86                            │
│                                                                              │
│  WHY NOT LARGER INSTANCES?                                                  │
│  • Diminishing returns: going from c6i.2xl to c6i.4xl saves ~2ms          │
│  • Network latency (to node, to Flashbots) dominates anyway               │
│  • Better to optimize code than throw hardware at it                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Decision 3: Why NOT Kubernetes/Serverless?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DEPLOYMENT MODEL: BARE EC2 vs K8S vs LAMBDA                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ❌ AWS Lambda / Serverless                                                 │
│  ─────────────────────────                                                  │
│  Why NOT:                                                                    │
│  • Cold starts: 100-500ms (unacceptable)                                   │
│  • No persistent WebSocket connections                                      │
│  • Can't maintain in-memory state                                          │
│  • Not designed for always-on, sub-100ms workloads                         │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  ⚠️  Kubernetes (EKS)                                                       │
│  ─────────────────────                                                       │
│  Why MAYBE NOT:                                                              │
│  • Overhead: Control plane adds latency                                     │
│  • Complexity: More moving parts = more failure modes                       │
│  • Pod scheduling: Unpredictable startup times                             │
│  • For 3-5 services, K8s is overkill                                       │
│                                                                              │
│  When K8s WOULD make sense:                                                 │
│  • Scaling to many trading strategies                                       │
│  • Multi-region deployment                                                  │
│  • Team >5 engineers                                                        │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  ✅ Bare EC2 with Systemd (OUR CHOICE)                                     │
│  ────────────────────────────────────                                       │
│  Why:                                                                        │
│  • Minimal overhead                                                          │
│  • Direct hardware access                                                    │
│  • Simple deployment: rsync + systemctl restart                            │
│  • Easy to reason about failure modes                                       │
│  • "Boring technology" = reliable technology                                │
│                                                                              │
│  Deployment script:                                                          │
│  ```bash                                                                     │
│  # deploy.sh                                                                │
│  rsync -avz ./dist/ ec2-user@arb-bot:/app/                                 │
│  ssh ec2-user@arb-bot "sudo systemctl restart arb-indexer"                 │
│  ssh ec2-user@arb-bot "sudo systemctl restart arb-detector"                │
│  ssh ec2-user@arb-bot "sudo systemctl restart arb-executor"                │
│  ```                                                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Decision 4: Inter-Service Communication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  HOW COMPONENTS TALK TO EACH OTHER                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  OPTIONS CONSIDERED:                                                        │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │  Option        │ Latency │ Complexity │ Verdict                    │   │
│  │────────────────────────────────────────────────────────────────────│   │
│  │  HTTP/REST     │ 1-5ms   │ Low        │ ❌ Too slow for hot path   │   │
│  │  gRPC          │ 0.5-2ms │ Medium     │ ⚠️  Good, but overhead     │   │
│  │  Redis Pub/Sub │ 0.1-1ms │ Low        │ ✅ Fast, simple            │   │
│  │  Shared Memory │ <0.1ms  │ High       │ ⚠️  Only if same machine   │   │
│  │  In-Process    │ ~0ms    │ Medium     │ ✅ Best for tight loops    │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  OUR APPROACH: HYBRID                                                       │
│                                                                              │
│  1. Hot Path (Indexer → Detector → Executor):                              │
│     • In-process OR Redis Pub/Sub                                          │
│     • Could even be single process with worker threads                     │
│     • Latency budget: <5ms total                                           │
│                                                                              │
│  2. State Storage (Pool States):                                           │
│     • Redis (shared across components)                                      │
│     • Enables multiple detector instances                                   │
│                                                                              │
│  3. Non-Critical Path (Logging, Metrics):                                  │
│     • Async HTTP/gRPC                                                       │
│     • Fire-and-forget, don't block hot path                                │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  SIMPLIFIED ARCHITECTURE (Single Process):                                 │
│                                                                              │
│  For a small team, consider running everything in ONE process:             │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐      │
│  │                    arb-bot (single binary)                       │      │
│  │                                                                  │      │
│  │   ┌───────────┐    ┌───────────┐    ┌───────────────────┐      │      │
│  │   │ Indexer   │───▶│ Detector  │───▶│ Executor          │      │      │
│  │   │ (thread)  │    │ (thread)  │    │ (thread)          │      │      │
│  │   └───────────┘    └───────────┘    └───────────────────┘      │      │
│  │         │                │                    │                 │      │
│  │         └────────────────┴────────────────────┘                 │      │
│  │                          │                                      │      │
│  │                    Shared Memory                                │      │
│  │                  (lock-free queues)                             │      │
│  │                                                                  │      │
│  └─────────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  Pros: Zero network latency between components                             │
│  Cons: Single point of failure, harder to scale individual parts          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Tech Stack Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  RECOMMENDED TECH STACK                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  LANGUAGE: TypeScript (Node.js) or Rust                                    │
│  ──────────────────────────────────────                                     │
│                                                                              │
│  TypeScript Pros:                      │ Rust Pros:                         │
│  • Fast development                    │ • Maximum performance              │
│  • Rich Ethereum libraries (ethers,    │ • Memory safety                    │
│    viem)                               │ • Predictable latency (no GC)      │
│  • Easy to hire for                    │ • Better for production HFT        │
│  • Good enough for <10ms latency       │                                    │
│                                        │                                    │
│  TypeScript Cons:                      │ Rust Cons:                         │
│  • GC pauses (mitigable)               │ • Slower development               │
│  • Not as fast as native               │ • Steeper learning curve           │
│  • V8 memory overhead                  │ • Fewer ready libraries            │
│                                                                              │
│  OUR CHOICE: TypeScript (with option to port hot paths to Rust later)     │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  FULL STACK:                                                                │
│                                                                              │
│  │ Layer          │ Technology        │ Why                               │
│  │────────────────────────────────────────────────────────────────────────│
│  │ Language       │ TypeScript 5.x    │ Balance of speed & productivity   │
│  │ Runtime        │ Node.js 20 LTS    │ Stable, good perf, native fetch   │
│  │ Ethereum       │ viem              │ Type-safe, modern, fast           │
│  │ State Store    │ Redis 7.x         │ Sub-ms reads, pub/sub             │
│  │ Database       │ PostgreSQL 15     │ Trade history, analytics          │
│  │ Metrics        │ Prometheus        │ Industry standard                 │
│  │ Dashboards     │ Grafana           │ Visualization                     │
│  │ Logs           │ Loki              │ Pairs well with Grafana           │
│  │ Alerting       │ PagerDuty         │ On-call rotations                 │
│  │ IaC            │ Terraform         │ Reproducible infrastructure       │
│  │ CI/CD          │ GitHub Actions    │ Simple, integrated                │
│  │ Secrets        │ AWS Secrets Mgr   │ Secure key storage                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Monthly Cost Estimate

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  INFRASTRUCTURE COST BREAKDOWN                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Component                    │ Spec              │ Monthly Cost            │
│  ──────────────────────────────────────────────────────────────────────    │
│  Dedicated Erigon Node        │ i3.2xlarge        │ $450                   │
│  (or bare metal equivalent)   │ 2TB NVMe, 64GB    │                        │
│  ──────────────────────────────────────────────────────────────────────    │
│  Compute (3x c6i instances)   │ c6i.xlarge x2     │ $280                   │
│                               │ c6i.2xlarge x1    │                        │
│  ──────────────────────────────────────────────────────────────────────    │
│  Redis (ElastiCache)          │ r6g.large         │ $95                    │
│  ──────────────────────────────────────────────────────────────────────    │
│  Backup RPC (Alchemy)         │ Growth plan       │ $200                   │
│  ──────────────────────────────────────────────────────────────────────    │
│  Monitoring (Grafana Cloud)   │ Pro plan          │ $50                    │
│  ──────────────────────────────────────────────────────────────────────    │
│  PostgreSQL (RDS)             │ db.t3.medium      │ $60                    │
│  ──────────────────────────────────────────────────────────────────────    │
│  Networking, misc             │                   │ $50                    │
│  ──────────────────────────────────────────────────────────────────────    │
│                                                                              │
│  TOTAL INFRASTRUCTURE                             │ ~$1,185/month          │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  BREAK-EVEN ANALYSIS:                                                       │
│  • Monthly infra cost: $1,185                                              │
│  • Average profit per successful arb: $100 (conservative)                  │
│  • Break-even: 12 successful arbs per month                                │
│  • Target: 50+ arbs/month → $5K+ profit → 4x+ ROI on infra                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Security & Risk Management

### Threat Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  WHAT CAN GO WRONG (AND HOW WE PREVENT IT)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  THREAT 1: Private Key Compromise                                          │
│  ─────────────────────────────────                                          │
│  Impact: CATASTROPHIC - All funds stolen                                   │
│                                                                              │
│  Mitigations:                                                                │
│  ├─ Store keys in AWS Secrets Manager or HSM                               │
│  ├─ Never log or expose keys in code                                       │
│  ├─ Use separate hot wallet with limited funds                             │
│  ├─ Main treasury in cold storage, auto-refill hot wallet                  │
│  └─ Enable AWS CloudTrail for access auditing                              │
│                                                                              │
│  Architecture:                                                               │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐             │
│  │ Cold Wallet  │─────▶│  Hot Wallet  │─────▶│  Arb Bot     │             │
│  │ (Multisig)   │      │  (Single Sig)│      │  (Executor)  │             │
│  │ $900K        │      │  $50K max    │      │              │             │
│  └──────────────┘      └──────────────┘      └──────────────┘             │
│        │                     │                                              │
│        │ Manual refill       │ Auto-refill when < $10K                     │
│        │ (requires 2/3 sig)  │ (capped at $50K)                            │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  THREAT 2: Smart Contract Bugs                                             │
│  ─────────────────────────────                                              │
│  Impact: HIGH - Funds locked or stolen via contract exploit                │
│                                                                              │
│  Mitigations:                                                                │
│  ├─ Keep contract minimal (less code = less bugs)                          │
│  ├─ Professional audit before mainnet                                       │
│  ├─ Extensive testing on forked mainnet                                    │
│  ├─ Pausable contract with owner emergency stop                            │
│  └─ Start with small amounts, increase gradually                           │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  THREAT 3: Execution Bugs (Bad Trades)                                     │
│  ─────────────────────────────────────                                      │
│  Impact: MEDIUM - Lose money on unprofitable trades                        │
│                                                                              │
│  Mitigations:                                                                │
│  ├─ Simulation before every trade (eth_call)                               │
│  ├─ minProfit check in smart contract (reverts if not met)                 │
│  ├─ Position limits (max trade size)                                       │
│  ├─ Daily loss limits (pause if exceeded)                                  │
│  └─ Extensive logging for post-mortem                                      │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  THREAT 4: Infrastructure Failure                                          │
│  ─────────────────────────────────                                          │
│  Impact: MEDIUM - Miss opportunities, stuck transactions                   │
│                                                                              │
│  Mitigations:                                                                │
│  ├─ Multiple RPC endpoints (failover)                                      │
│  ├─ Health checks with auto-restart                                        │
│  ├─ Nonce management to handle stuck txs                                   │
│  └─ Alerting for any component failure                                     │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  THREAT 5: MEV Attacks (Sandwich/Frontrun)                                 │
│  ─────────────────────────────────────────                                  │
│  Impact: MEDIUM - Profit stolen by attackers                               │
│                                                                              │
│  Mitigations:                                                                │
│  ├─ MANDATORY: Use Flashbots/private mempool                               │
│  ├─ Never submit to public mempool                                         │
│  └─ Use MEV-Share to get kickbacks if we are sandwiched                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Circuit Breakers

```
┌────────────────────────────────────────────────────────────────────────────┐
│  AUTOMATED SAFETY CONTROLS                                                 │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  CIRCUIT BREAKER                │  TRIGGER              │ ACTION    │   │
│  │─────────────────────────────────────────────────────────────────────│   │
│  │  Daily Loss Limit               │  Losses > $1,000/day  │ HALT      │   │
│  │  Consecutive Failures           │  3 failed txs in row  │ PAUSE 5m  │   │
│  │  Gas Price Spike                │  Gas > 500 gwei       │ PAUSE     │   │
│  │  Low Hot Wallet Balance         │  Balance < $5,000     │ ALERT     │   │
│  │  Node Sync Lag                  │  >3 blocks behind     │ HALT      │   │
│  │  High Revert Rate               │  >20% reverts/hour    │ PAUSE 15m │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Monitoring & Observability

### Key Metrics Dashboard

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  METRICS WE TRACK (Prometheus + Grafana)                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  BUSINESS METRICS (Most Important)                                         │
│  ─────────────────────────────────                                          │
│  • arb_profit_total            - Cumulative profit in USD                  │
│  • arb_trades_total            - Counter of executed trades                │
│  • arb_trade_success_rate      - % of profitable trades                    │
│  • arb_opportunities_detected  - Opportunities found per minute            │
│  • arb_opportunities_executed  - Opportunities acted on                    │
│  • arb_gas_spent_total         - Total gas costs                           │
│                                                                              │
│  LATENCY METRICS (Performance)                                             │
│  ─────────────────────────────                                              │
│  • block_to_detection_ms       - Time from block to opportunity found      │
│  • detection_to_submission_ms  - Time to submit transaction                │
│  • end_to_end_latency_ms       - Total pipeline latency                    │
│  • rpc_request_duration_ms     - Node query latency                        │
│                                                                              │
│  SYSTEM HEALTH                                                              │
│  ─────────────                                                              │
│  • hot_wallet_balance_eth      - Current trading capital                   │
│  • node_block_height           - Are we synced?                            │
│  • redis_connection_status     - Cache health                              │
│  • component_uptime_seconds    - Per-service uptime                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Example Grafana Dashboard Layout

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ARBITRAGE BOT DASHBOARD                                         [24h ▼]   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌───────────┐ │
│  │  TODAY'S P&L    │ │  WIN RATE       │ │  TRADES TODAY   │ │  UPTIME   │ │
│  │                 │ │                 │ │                 │ │           │ │
│  │    +$847.32     │ │     87.5%       │ │       24        │ │  99.97%   │ │
│  │    ▲ $123.50    │ │    (21/24)      │ │   ▲ 8 vs avg    │ │           │ │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘ └───────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │  CUMULATIVE PROFIT (7 DAYS)                                            ││
│  │     $│                                                    ╭────        ││
│  │  5000│                                              ╭─────╯            ││
│  │  4000│                                    ╭─────────╯                  ││
│  │  3000│                          ╭─────────╯                            ││
│  │  2000│                ╭─────────╯                                      ││
│  │  1000│      ╭─────────╯                                                ││
│  │      └──────────────────────────────────────────────────────────────  ││
│  │        Mon     Tue     Wed     Thu     Fri     Sat     Sun             ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌──────────────────────────────────┐ ┌──────────────────────────────────┐ │
│  │  LATENCY DISTRIBUTION (p50/p95)  │ │  OPPORTUNITIES vs EXECUTED       │ │
│  │                                  │ │                                  │ │
│  │  Block→Detection:  8ms / 15ms   │ │  ████████████░░░░  145 / 24     │ │
│  │  Detection→Submit: 12ms / 25ms  │ │                    (16.5% hit)   │ │
│  │  End-to-End:       23ms / 45ms  │ │                                  │ │
│  └──────────────────────────────────┘ └──────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │  RECENT TRADES                                                         ││
│  │──────────────────────────────────────────────────────────────────────  ││
│  │  Time     │ Path              │ Size    │ Profit  │ Gas    │ Status   ││
│  │  14:32:01 │ ETH→USDC→ETH     │ $25,000 │ +$127   │ $18    │ ✅       ││
│  │  14:28:45 │ WBTC→ETH→WBTC    │ $15,000 │ +$84    │ $15    │ ✅       ││
│  │  14:25:12 │ ETH→DAI→ETH      │ $10,000 │ -$22    │ $22    │ ❌ revert││
│  │  14:21:33 │ LINK→ETH→LINK    │ $8,000  │ +$45    │ $12    │ ✅       ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Alerting Rules

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PAGERDUTY ALERT CONFIGURATION                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SEVERITY: CRITICAL (Wake up at 3am)                                       │
│  ─────────────────────────────────────                                      │
│  • hot_wallet_balance < $2,000                                             │
│  • daily_loss > $1,000                                                     │
│  • all_components_down for > 1 minute                                      │
│  • smart_contract_paused (someone hit emergency stop)                      │
│                                                                              │
│  SEVERITY: HIGH (Respond within 1 hour)                                    │
│  ──────────────────────────────────────                                     │
│  • win_rate < 50% over last 2 hours                                        │
│  • node_sync_lag > 3 blocks                                                │
│  • consecutive_failures >= 3                                               │
│                                                                              │
│  SEVERITY: WARNING (Check next business day)                               │
│  ────────────────────────────────────────────                               │
│  • gas_prices > 200 gwei sustained 1 hour                                  │
│  • opportunity_detection_rate < 10/hour (market might be quiet)            │
│  • disk_usage > 80%                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Mainnet Testing & Profitability Metrics

### Testing Strategy on Mainnet

Before deploying with full capital, we run a **gradual rollout** with increasing capital and comprehensive metrics tracking to ensure profitability and catch issues early.

**Phased Approach:**
1. **Week 1**: $1,000 capital, monitor all metrics
2. **Week 2**: $5,000 capital (if Week 1 profitable)
3. **Week 3**: $25,000 capital (if Week 2 profitable)
4. **Week 4+**: Full capital deployment (if consistently profitable)

---

### Critical Profitability Metrics

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PROFITABILITY TRACKING (Must Track Daily)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PRIMARY METRICS                                                            │
│  ──────────────                                                             │
│  • net_profit_usd              - Total profit after ALL costs             │
│  • gross_profit_usd             - Profit before gas/fees                    │
│  • total_gas_cost_usd           - All gas spent (successful + failed)      │
│  • total_fees_paid_usd          - DEX fees + flash loan fees               │
│  • roi_percentage               - (net_profit / capital_deployed) × 100    │
│  • profit_per_trade_avg         - Average net profit per successful trade  │
│                                                                              │
│  COST BREAKDOWN                                                             │
│  ─────────────                                                                │
│  • gas_successful_txs           - Gas for profitable trades                 │
│  • gas_failed_txs               - Gas for reverted transactions (WASTED)   │
│  • gas_replacement_txs          - Gas for replacing stuck transactions     │
│  • flashbots_tips_total          - Total tips paid to Flashbots             │
│  • dex_fees_total                - Total 0.3% fees paid to pools           │
│  • flash_loan_fees_total         - Fees if using flash loans               │
│                                                                              │
│  EFFICIENCY METRICS                                                         │
│  ────────────────                                                            │
│  • gas_efficiency_ratio         - (gas_successful / gas_total) × 100       │
│  • cost_per_opportunity          - Total costs / opportunities_detected    │
│  • profit_margin_percentage      - (net_profit / gross_profit) × 100       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Failed Transaction Analysis

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  FAILED TRANSACTION TRACKING                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  FAILURE METRICS                                                            │
│  ──────────────                                                             │
│  • failed_tx_count              - Total reverted transactions               │
│  • failed_tx_gas_cost            - Gas wasted on failed txs                 │
│  • failure_rate                  - (failed / total_submitted) × 100        │
│  • consecutive_failures          - Current streak of failures                │
│                                                                              │
│  FAILURE REASONS (Categorize each failure)                                 │
│  ────────────────────────────────────────                                   │
│  • slippage_exceeded_count      - Price moved too much                      │
│  • insufficient_liquidity_count - Pool didn't have enough tokens            │
│  • frontrun_count                - Someone executed before us               │
│  • state_changed_count           - Pool state changed (stale data)          │
│  • gas_estimation_error_count    - Gas limit too low                        │
│  • smart_contract_error_count    - Bug in our contract                      │
│  • unknown_revert_count          - Other reasons                            │
│                                                                              │
│  FAILURE PATTERNS                                                            │
│  ───────────────                                                             │
│  • failures_by_pool             - Which pools fail most?                   │
│  • failures_by_time_of_day      - Are certain times worse?                 │
│  • failures_by_gas_price         - Correlation with gas prices?             │
│  • failures_by_trade_size        - Do larger trades fail more?              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Wasted Gas Tracking

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  GAS WASTE ANALYSIS                                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  GAS WASTE METRICS                                                           │
│  ────────────────                                                            │
│  • wasted_gas_total            - Gas spent on failed/replaced txs          │
│  • wasted_gas_percentage       - (wasted / total_gas) × 100                 │
│  • wasted_gas_cost_usd          - USD value of wasted gas                   │
│  • avg_gas_per_failed_tx        - Average gas per failure                   │
│                                                                              │
│  GAS WASTE BREAKDOWN                                                        │
│  ──────────────────                                                          │
│  • gas_wasted_failed_txs        - Gas on reverted transactions              │
│  • gas_wasted_replacements      - Gas on replacing stuck txs                │
│  • gas_wasted_over_estimation   - Gas limit set too high (unused)           │
│                                                                              │
│  GAS EFFICIENCY TARGETS                                                      │
│  ─────────────────────                                                        │
│  • Target: <10% wasted gas                                                    │
│  • Alert if: >20% wasted gas                                                  │
│  • Critical if: >30% wasted gas                                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Opportunity Quality Metrics

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  OPPORTUNITY ANALYSIS                                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  DETECTION METRICS                                                           │
│  ────────────────                                                            │
│  • opportunities_detected       - Total opportunities found                 │
│  • opportunities_executed       - Opportunities we acted on                 │
│  • opportunities_missed         - Detected but not executed                  │
│  • execution_rate               - (executed / detected) × 100               │
│                                                                              │
│  QUALITY METRICS                                                             │
│  ──────────────                                                              │
│  • avg_expected_profit          - Average profit we calculated               │
│  • avg_actual_profit            - Average profit we actually made           │
│  • profit_accuracy              - (actual / expected) × 100                  │
│  • false_positive_rate          - Opportunities that looked good but failed  │
│  • false_negative_rate          - Opportunities we skipped but were good     │
│                                                                              │
│  MISSED OPPORTUNITY ANALYSIS                                                │
│  ─────────────────────────                                                    │
│  • missed_due_to_capital       - Not enough capital available              │
│  • missed_due_to_latency        - Too slow, someone else got it             │
│  • missed_due_to_risk           - Below risk threshold                      │
│  • missed_due_to_cooldown       - In cooldown period                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Performance vs. Expectations

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  EXPECTED vs ACTUAL COMPARISON                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PROFIT COMPARISON                                                           │
│  ────────────────                                                            │
│  • expected_profit_total       - Sum of all expected profits               │
│  • actual_profit_total          - Sum of all actual profits                │
│  • profit_gap                   - Difference (expected - actual)            │
│  • profit_gap_percentage        - (gap / expected) × 100                   │
│                                                                              │
│  GAS COMPARISON                                                              │
│  ─────────────                                                                │
│  • expected_gas_total           - Sum of estimated gas                      │
│  • actual_gas_total             - Sum of actual gas used                    │
│  • gas_estimation_error         - Difference                                │
│  • gas_estimation_accuracy      - (expected / actual) × 100                  │
│                                                                              │
│  TRADE SIZE COMPARISON                                                       │
│  ────────────────────                                                        │
│  • expected_trade_size_avg      - Average size we calculated                │
│  • actual_trade_size_avg        - Average size we executed                  │
│  • size_difference               - Why different? (slippage, limits)         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Daily Health Checks

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DAILY METRICS TO REVIEW                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PROFITABILITY CHECKLIST                                                     │
│  ─────────────────────                                                       │
│  [ ] Net profit > $0 (profitable day)                                        │
│  [ ] Win rate > 70% (most trades succeed)                                    │
│  [ ] Gas efficiency > 90% (wasted gas < 10%)                                │
│  [ ] Profit accuracy > 80% (actual close to expected)                       │
│  [ ] No consecutive failures > 3                                            │
│                                                                              │
│  COST ANALYSIS                                                               │
│  ─────────────                                                                │
│  [ ] Gas costs < 20% of gross profit                                         │
│  [ ] Failed tx gas < 10% of total gas                                        │
│  [ ] Flashbots tips reasonable (< $50/day)                                   │
│  [ ] DEX fees accounted for correctly                                        │
│                                                                              │
│  OPPORTUNITY QUALITY                                                         │
│  ──────────────────                                                          │
│  [ ] Execution rate > 15% (not too selective)                               │
│  [ ] False positive rate < 20% (not too many bad trades)                    │
│  [ ] Average profit per trade > $50                                          │
│  [ ] No single pool causing > 30% of failures                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Alerting on Profitability Issues

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PROFITABILITY ALERTS                                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CRITICAL (Immediate Action Required)                                        │
│  ────────────────────────────────────                                       │
│  • Daily net profit < -$500 (losing money)                                   │
│  • Win rate < 40% for 2 hours (systemic issue)                              │
│  • Gas waste > 40% (burning money on failures)                               │
│  • Profit gap > 50% (calculation way off)                                    │
│                                                                              │
│  HIGH (Investigate Today)                                                   │
│  ───────────────────────                                                     │
│  • Daily net profit < $0 (unprofitable day)                                 │
│  • Win rate < 60% for 4 hours                                               │
│  • Gas waste > 25%                                                           │
│  • Consecutive failures >= 5                                                 │
│  • Single pool causing > 50% of failures                                     │
│                                                                              │
│  WARNING (Monitor Closely)                                                   │
│  ─────────────────────────                                                   │
│  • Profit accuracy < 70% (estimates off)                                     │
│  • Execution rate < 10% (too selective)                                     │
│  • Average profit per trade < $30                                           │
│  • Gas costs > 30% of gross profit                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Example Daily Report

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DAILY PROFITABILITY REPORT - Dec 15, 2024                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SUMMARY                                                                     │
│  ───────                                                                     │
│  Net Profit:        +$847.32 ✅                                             │
│  Gross Profit:      +$1,245.80                                               │
│  Total Costs:       -$398.48                                                 │
│  ROI:               0.85% (on $100K capital)                                 │
│                                                                              │
│  TRADES                                                                            │
│  ──────                                                                      │
│  Total Executed:    24                                                        │
│  Successful:        21 (87.5%) ✅                                            │
│  Failed:             3 (12.5%)                                               │
│  Avg Profit/Trade:  $40.35                                                   │
│                                                                              │
│  COSTS                                                                       │
│  ─────                                                                       │
│  Gas (Successful):  $285.30                                                 │
│  Gas (Failed):      $67.20  ⚠️  (wasted)                                    │
│  Flashbots Tips:    $32.50                                                   │
│  DEX Fees:          $13.48                                                   │
│  Gas Efficiency:    81% (19% wasted) ⚠️                                      │
│                                                                              │
│  OPPORTUNITIES                                                               │
│  ───────────                                                                 │
│  Detected:          145                                                       │
│  Executed:          24 (16.6%)                                               │
│  Missed (Capital):  8                                                        │
│  Missed (Latency):  12                                                       │
│  Missed (Risk):     101                                                      │
│                                                                              │
│  ACCURACY                                                                     │
│  ───────                                                                     │
│  Expected Profit:   $1,180.50                                                │
│  Actual Profit:     $1,245.80                                                │
│  Accuracy:          105.5% ✅ (better than expected!)                        │
│                                                                              │
│  ISSUES                                                                      │
│  ──────                                                                      │
│  • 3 failed transactions (slippage exceeded)                                │
│  • Gas waste at 19% (target: <10%) ⚠️                                        │
│  • One pool (ETH/USDC V2) caused 2/3 failures - investigate                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Metrics Dashboard (Testing Phase)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TESTING PHASE DASHBOARD                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐             │
│  │  NET PROFIT     │ │  WIN RATE       │ │  GAS EFFICIENCY │             │
│  │                 │ │                 │ │                 │             │
│  │  +$847.32      │ │     87.5%       │ │      81%        │             │
│  │  (Today)        │ │    (21/24)      │ │   ⚠️ Target:90% │             │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘             │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │  PROFIT BREAKDOWN                                                       ││
│  │     $│                                                                  ││
│  │ 1500 │  Gross Profit: $1,245.80                                        ││
│  │ 1000 │  ─────────────────────────────────────                         ││
│  │  500 │  Net Profit: $847.32                                            ││
│  │    0 │  ─────────────────────────────────────                         ││
│  │ -500 │  Costs: -$398.48                                               ││
│  │      └──────────────────────────────────────────────────────────────  ││
│  │        Gas: -$352.50  │  Tips: -$32.50  │  Fees: -$13.48              ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌──────────────────────────────────┐ ┌──────────────────────────────────┐ │
│  │  FAILED TRANSACTIONS             │ │  GAS WASTE BREAKDOWN              │ │
│  │                                  │ │                                  │ │
│  │  Total: 3                        │ │  Successful: $285.30 (81%)       │ │
│  │  ─────────────────────────────── │ │  Failed: $67.20 (19%) ⚠️        │ │
│  │  Slippage: 2                     │ │                                  │ │
│  │  Frontrun: 1                     │ │  Target: <10% wasted              │ │
│  │  Liquidity: 0                    │ │                                  │ │
│  └──────────────────────────────────┘ └──────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │  EXPECTED vs ACTUAL PROFIT                                             ││
│  │                                                                         ││
│  │  Expected: $1,180.50                                                   ││
│  │  Actual:   $1,245.80                                                   ││
│  │  Gap:      +$65.30 (5.5% better than expected) ✅                      ││
│  │                                                                         ││
│  │  This indicates our profit calculation is conservative (good!)         ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Testing Phase Success Criteria

**Before scaling to full capital, we must achieve:**

1. **Profitability**: 7 consecutive days of positive net profit
2. **Win Rate**: >75% over 7 days
3. **Gas Efficiency**: <15% wasted gas over 7 days
4. **Profit Accuracy**: Actual within 20% of expected
5. **No Critical Issues**: No days with >$500 loss
6. **Stability**: No crashes, no stuck transactions

**If any criteria not met:**
- Pause scaling
- Investigate root cause
- Fix issues
- Restart testing phase

---

## 13. Future Considerations

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  WHAT WE'D ADD IN V2                                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. MULTI-CHAIN EXPANSION                                                  │
│     • Arbitrum, Optimism, Base (lower gas, more opportunities)             │
│     • Cross-chain arbitrage via bridges                                    │
│                                                                              │
│  2. FLASH LOANS                                                             │
│     • Borrow capital for trade, repay in same tx                           │
│     • Enables larger trades without capital lockup                         │
│     • Risk: Higher complexity, more failure modes                          │
│                                                                              │
│  3. MORE DEX SUPPORT                                                        │
│     • SushiSwap, Curve, Balancer                                           │
│     • More pools = more opportunities                                       │
│                                                                              │
│  4. MACHINE LEARNING                                                        │
│     • Predict gas prices for better timing                                 │
│     • Predict which opportunities will succeed                             │
│     • Optimize trade sizing dynamically                                    │
│                                                                              │
│  5. BACKTESTING INFRASTRUCTURE                                             │
│     • Replay historical blocks to test strategies                          │
│     • Measure theoretical vs actual performance                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 13. Summary

This design document outlines a production-ready arbitrage bot system with:

| Aspect | Approach |
|--------|----------|
| **Latency** | <50ms end-to-end via dedicated nodes, in-memory state, optimized code paths |
| **Reliability** | Circuit breakers, fallback RPCs, comprehensive monitoring |
| **Security** | Private mempools (Flashbots), hot/cold wallet separation, minimal smart contract |
| **Profitability** | Accurate profit calculation including all costs, optimal trade sizing |
| **Observability** | Prometheus metrics, Grafana dashboards, PagerDuty alerting |
| **Cost** | ~$1,200/month infrastructure, break-even at 12 trades/month |

**Key Insight**: The hardest part isn't finding arbitrage opportunities - it's executing them profitably after accounting for gas, fees, slippage, and competition. This system is designed to be conservative: better to miss a marginal opportunity than to execute an unprofitable trade.

---
