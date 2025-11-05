# LayerZero/Stargate Data Provider Plugin

Production-ready plugin for collecting cross-chain bridge metrics from LayerZero and Stargate protocols.

## Provider Overview

**Provider**: LayerZero + Stargate  
**Data Sources**: Official off-chain APIs (no on-chain simulation)

### Why LayerZero + Stargate?

- **LayerZero** is the omnichain messaging protocol
- **Stargate** is the primary liquidity bridge built on LayerZero
- Most asset transfers and liquidity occur via Stargate
- We use **LayerZero Scan** for volume metrics and **Stargate API** for transfer economics

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    LayerZero Plugin                         │
├─────────────────────────────────────────────────────────────┤
│  Volume Metrics      │  LayerZero Scan API                  │
│  • 24h/7d/30d volume │  GET /messages/latest (paginated)    │
├─────────────────────────────────────────────────────────────┤
│  Rates & Fees        │  Stargate API                        │
│  • Exchange rates    │  GET /quotes                         │
│  • Fee breakdown     │  • Returns srcAmount → dstAmount     │
│                      │  • Includes fees[] array             │
├─────────────────────────────────────────────────────────────┤
│  Liquidity Depth     │  Stargate API (Binary Search)        │
│  • 50bps threshold   │  Probe /quotes with increasing       │
│  • 100bps threshold  │  amounts to find max size            │
├─────────────────────────────────────────────────────────────┤
│  Available Assets    │  Stargate API                        │
│  • Token list        │  GET /tokens, GET /chains            │
│  • Chain mapping     │  Map chainKey → chainId              │
└─────────────────────────────────────────────────────────────┘
```

## Quick Start

```bash
# Install dependencies
bun install

# Run tests
bun test

# Build
bun run build

# Development mode
bun run dev
```

## Official Documentation

### Protocol Documentation
- **LayerZero Protocol**: https://docs.layerzero.network/
- **Stargate Protocol**: https://stargateprotocol.gitbook.io/stargate/
- **LayerZero Scan**: https://layerzeroscan.com/

### API Documentation
- **LayerZero Scan API**: https://scan.layerzero-api.com/v1
  - Documentation: https://docs.layerzero.network/contracts/layerzero-scan
- **Stargate API**: https://stargate.finance/api/v1
  - Documentation: https://stargateprotocol.gitbook.io/stargate/developers/api

## API Endpoints Used

### 1. LayerZero Scan API

**Base URL**: `https://scan.layerzero-api.com/v1`  
**Documentation**: https://docs.layerzero.network/contracts/layerzero-scan

- **GET /messages/latest** - Fetch cross-chain messages with pagination
  - Query params: `start` (ISO datetime), `end` (ISO datetime), `limit`, `nextToken`
  - Returns: Message data including `source.tx.value` (native fee/value)
  - Used for: Volume aggregation across time windows
  - **Note**: `source.tx.value` represents native gas fees, not transfer amounts

### 2. Stargate API

**Base URL**: `https://stargate.finance/api/v1`  
**Documentation**: https://stargateprotocol.gitbook.io/stargate/developers/api

- **GET /quotes** - Get transfer quote for a route
  - Params: `srcToken`, `dstToken`, `srcChainKey`, `dstChainKey`, `srcAmount`, etc.
  - Returns: `srcAmount`, `dstAmount`, `fees[]` array
  - Used for: Rates, fees, and liquidity depth probing

- **GET /tokens** - List all supported tokens
  - Returns: Array of tokens with `chainKey`, `address`, `symbol`, `decimals`
  - Used for: Asset listings

- **GET /chains** - List all supported chains
  - Returns: Array of chains with `chainKey`, `chainId`, `name`
  - Used for: Chain ID mapping

## Four Required Metrics

### 1. Volume

**Source**: LayerZero Scan `/messages/latest`

**Method**: Sum `source.tx.value` across all messages in time window

**Important Limitation**: 
- `source.tx.value` represents **native gas fees**, NOT transfer amounts
- This is a limitation of the LayerZero Scan API
- Actual transfer amounts are embedded in message payloads (not easily parseable)

**Normalization**: Raw values in native smallest units → USD estimate (ETH price × value / 1e18)

**Time Windows**: 24h, 7d, 30d

**Alternative Approaches** (if available):
- Parse Stargate-specific message payloads for actual amounts
- Use Stargate analytics/subgraph (if exists)
- Filter messages by Stargate app ID

```typescript
// Example output
{
  window: "24h",
  volumeUsd: 1234567.89,
  measuredAt: "2024-01-15T12:00:00Z"
}
```

### 2. Rates (Fees)

**Source**: Stargate `/quotes`

**Method**: 
- Call `/quotes` for each route/notional combination
- Calculate `effectiveRate = (dstAmount / 10^dstDec) / (srcAmount / 10^srcDec)`
- Sum `fees[]` array for total fees

**Output**: Exchange rate and fee breakdown per route

```typescript
// Example output
{
  source: { chainId: "1", assetId: "0x...", symbol: "USDC", decimals: 6 },
  destination: { chainId: "137", assetId: "0x...", symbol: "USDC", decimals: 6 },
  amountIn: "1000000",  // 1 USDC
  amountOut: "995000",  // 0.995 USDC
  effectiveRate: 0.995,
  totalFeesUsd: 0.005,
  quotedAt: "2024-01-15T12:00:00Z"
}
```

### 3. Liquidity Depth

**Source**: Stargate `/quotes` (binary search)

**Method**:
1. Get baseline quote with 1 unit to establish base rate
2. Binary search over amounts to find max size where slippage ≤ target
3. Slippage = `(1 - currentRate / baseRate) × 10,000` basis points
4. Find thresholds for 50bps (0.5%) and 100bps (1.0%)

**Why Binary Search**:
- Stargate doesn't expose a direct liquidity depth API
- Binary search efficiently finds max amount (~24 iterations per threshold)
- Quotes reflect actual pool liquidity and price impact

**Limitations**:
- Requires ~48 API calls per route (24 per threshold)
- Rate-limited to 3 rps = ~16 seconds per route minimum
- Assumes quote slippage accurately reflects pool depth

**Output**: Max notional at each slippage threshold

```typescript
// Example output
{
  route: { source: {...}, destination: {...} },
  thresholds: [
    { maxAmountIn: "500000000000", slippageBps: 50 },   // $500K at 0.5%
    { maxAmountIn: "1000000000000", slippageBps: 100 }  // $1M at 1.0%
  ],
  measuredAt: "2024-01-15T12:00:00Z"
}
```

### 4. Available Assets

**Source**: Stargate `/tokens` + `/chains`

**Method**:
- Fetch all tokens from `/tokens`
- Fetch chain metadata from `/chains`
- Map `chainKey` → `chainId` for standardization
- Return normalized asset list

```typescript
// Example output
{
  assets: [
    { chainId: "1", assetId: "0xa0b86...", symbol: "USDC", decimals: 6 },
    { chainId: "137", assetId: "0x2791...", symbol: "USDC", decimals: 6 },
    ...
  ],
  measuredAt: "2024-01-15T12:00:00Z"
}
```

## Environment Variables

```bash
# LayerZero Scan API
LZ_SCAN_BASE_URL=https://scan.layerzero-api.com/v1

# Stargate API
STARGATE_BASE_URL=https://stargate.finance/api/v1

# HTTP Configuration
HTTP_TIMEOUT_MS=12000
MAX_RETRIES=4

# Rate Limiting (requests per second)
RATE_LIMIT_RPS_LZ=3
RATE_LIMIT_RPS_STG=3
```

**Note**: No API keys required - both APIs are publicly accessible.

## Testing

### Quick Test Route

Test with USDC Ethereum → Polygon:

```typescript
{
  source: {
    chainId: "1",
    assetId: "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
    symbol: "USDC",
    decimals: 6
  },
  destination: {
    chainId: "137",
    assetId: "0x3c499c542cef5e3811e1192ce70d8cc03d5c3359",
    symbol: "USDC",
    decimals: 6
  }
}
```

### Run Tests

```bash
# All tests
bun test

# Unit tests only
bun run test

# Integration tests
bun run test:integration

# Watch mode
bun run test:watch
```

## Implementation Details

### Retry Logic

Exponential backoff with jitter:
- Formula: `backoff = base × 2^attempt + random(0, base)`
- Max delay: 3 seconds
- Max retries: 4 attempts
- Retries on: 429 (rate limit) and 5xx errors

### Rate Limiting

Token bucket algorithm:
- Separate limiters for LayerZero Scan (3 rps) and Stargate (3 rps)
- Tokens refill continuously
- Requests block until token available

### Decimal Normalization

All amounts stored as strings in smallest units:
- USDC (6 decimals): `"1000000"` = 1 USDC
- ETH (18 decimals): `"1000000000000000000"` = 1 ETH

Effective rate calculation normalizes for decimals:
```typescript
effectiveRate = (dstAmount / 10^dstDecimals) / (srcAmount / 10^srcDecimals)
```

### Chain ID Mapping

Simplified mapping (production should use comprehensive registry):

| Chain ID | Chain Key  | Network    |
|----------|------------|------------|
| 1        | ethereum   | Ethereum   |
| 137      | polygon    | Polygon    |
| 42161    | arbitrum   | Arbitrum   |
| 10       | optimism   | Optimism   |
| 56       | bsc        | BSC        |
| 43114    | avalanche  | Avalanche  |
| 8453     | base       | Base       |

## Project Structure

```
layerzero-plugin/
├── src/
│   ├── client/
│   │   ├── lzScan.ts       # LayerZero Scan API client
│   │   └── stargate.ts     # Stargate API client
│   ├── services/
│   │   └── index.ts        # Core business logic (volume, rates, liquidity)
│   ├── utils/
│   │   ├── retry.ts        # Exponential backoff with jitter
│   │   └── rateLimit.ts    # Token bucket rate limiter
│   ├── contract.ts         # oRPC contract definition
│   ├── service.ts          # Main service orchestration
│   ├── index.ts            # Plugin entry point
│   └── env.ts              # Environment config schema
├── __tests__/
│   ├── unit/               # Unit tests
│   ├── integration/        # Integration tests
│   └── fixtures/           # Test fixtures
└── README.md
```

## Key Design Decisions

### Why LayerZero Scan for Volume?

LayerZero Scan provides neutral message-level data across all LayerZero applications. The `source.tx.value` field captures the native value/fee associated with each message, giving us a protocol-wide volume metric.

### Why Stargate for Rates/Liquidity?

Stargate is where most asset transfers occur. Its `/quotes` endpoint:
- Returns accurate fees that vary by route and gas conditions
- Provides actual `dstAmount` accounting for slippage and fees
- Can be probed to discover liquidity depth

### Why Binary Search for Liquidity?

Stargate doesn't expose a direct liquidity depth API. Binary search on quotes:
- Finds the exact threshold where slippage exceeds target
- Requires ~24 API calls per threshold (logarithmic)
- Gives accurate, real-time liquidity data

## Caveats & Future Improvements

### Current Limitations

1. **Volume USD Conversion**: Uses rough ETH price estimate. Production should fetch real-time prices.
2. **Chain Mapping**: Simplified mapping. Production needs comprehensive chain registry.
3. **OFT Assets**: Doesn't merge LayerZero OFT list. Could enrich asset data.
4. **Parallel Limits**: Sequential route processing. Could parallelize with concurrency limits.

### Production Enhancements

- Add price oracle integration for accurate USD conversion
- Implement comprehensive chain/token registry
- Add metrics/observability (Prometheus, DataDog)
- Cache frequently accessed data (tokens, chains)
- Add circuit breaker for API failures
- Implement request deduplication

## License

Part of the NEAR Intents data collection system.
