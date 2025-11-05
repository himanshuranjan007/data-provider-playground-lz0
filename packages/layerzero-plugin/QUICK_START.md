# LayerZero Plugin - Quick Start Guide

## 🚀 Get Started in 5 Minutes

### Step 1: Install Dependencies

```bash
cd packages/layerzero-plugin
bun install
```

### Step 2: Build the Plugin

```bash
bun run build
```

### Step 3: Run Tests

```bash
# Run all tests
bun test

# Run with coverage
bun run coverage
```

## 🧪 Testing with Real APIs

### Example 1: Test USDC Ethereum → Polygon

Create a test file `test-real-api.ts`:

```typescript
import { DataProviderService } from './src/service';
import { EnvSchema } from './src/env';

async function testRealAPI() {
  // Initialize service
  const config = EnvSchema.parse({
    LZ_SCAN_BASE_URL: 'https://scan.layerzero-api.com/v1',
    STARGATE_BASE_URL: 'https://stargate.finance/api/v1',
    HTTP_TIMEOUT_MS: 12000,
    MAX_RETRIES: 4,
    RATE_LIMIT_RPS_LZ: 3,
    RATE_LIMIT_RPS_STG: 3,
  });

  const service = new DataProviderService(config);

  // Test route: USDC Ethereum → Polygon
  const route = {
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
  };

  console.log('🔍 Fetching snapshot...\n');

  const snapshot = await service.getSnapshot({
    routes: [route],
    notionals: ["1000000"], // 1 USDC
    includeWindows: ["24h"]
  });

  console.log('✅ Snapshot received!\n');
  console.log('📊 Volumes:', snapshot.volumes);
  console.log('💱 Rates:', snapshot.rates);
  console.log('💧 Liquidity:', snapshot.liquidity);
  console.log('🪙 Assets:', snapshot.listedAssets.assets.length, 'total');
}

testRealAPI().catch(console.error);
```

Run it:

```bash
bun run test-real-api.ts
```

### Example 2: Test Multiple Routes

```typescript
const routes = [
  {
    source: { chainId: "1", assetId: "0xa0b86...", symbol: "USDC", decimals: 6 },
    destination: { chainId: "137", assetId: "0x3c499...", symbol: "USDC", decimals: 6 }
  },
  {
    source: { chainId: "1", assetId: "0xa0b86...", symbol: "USDC", decimals: 6 },
    destination: { chainId: "42161", assetId: "0xaf88...", symbol: "USDC", decimals: 6 }
  }
];

const snapshot = await service.getSnapshot({
  routes,
  notionals: ["1000000", "10000000"], // 1 USDC, 10 USDC
  includeWindows: ["24h", "7d"]
});
```

## 🔍 Debugging

### Enable Verbose Logging

The service logs to console. Check for:

```
[LayerZero] Fetching snapshot for 1 routes
Failed to fetch volume for 24h: <error>
Failed to fetch rate for route: <error>
```

### Common Issues

#### 1. Rate Limiting

**Symptom**: 429 errors

**Solution**: Increase retry delay or decrease RPS:

```bash
RATE_LIMIT_RPS_LZ=2
RATE_LIMIT_RPS_STG=2
```

#### 2. Timeout Errors

**Symptom**: Request timeout

**Solution**: Increase timeout:

```bash
HTTP_TIMEOUT_MS=20000
```

#### 3. Invalid Chain/Token

**Symptom**: "No quotes available"

**Solution**: Verify chain IDs and token addresses match Stargate's supported assets.

### Check API Availability

Test LayerZero Scan:

```bash
curl "https://scan.layerzero-api.com/v1/messages/latest?limit=1"
```

Test Stargate:

```bash
curl "https://stargate.finance/api/v1/chains"
curl "https://stargate.finance/api/v1/tokens"
```

## 📊 Understanding the Output

### Volume Output

```json
{
  "window": "24h",
  "volumeUsd": 1234567.89,
  "measuredAt": "2024-01-15T12:00:00Z"
}
```

- `volumeUsd`: Estimated USD volume (uses rough ETH price)
- Based on LayerZero message values

### Rate Output

```json
{
  "source": { "chainId": "1", "assetId": "0x...", "symbol": "USDC", "decimals": 6 },
  "destination": { "chainId": "137", "assetId": "0x...", "symbol": "USDC", "decimals": 6 },
  "amountIn": "1000000",
  "amountOut": "995000",
  "effectiveRate": 0.995,
  "totalFeesUsd": 0.005,
  "quotedAt": "2024-01-15T12:00:00Z"
}
```

- `effectiveRate`: Decimal-normalized exchange rate
- `totalFeesUsd`: Sum of all fees from Stargate

### Liquidity Output

```json
{
  "route": { "source": {...}, "destination": {...} },
  "thresholds": [
    { "maxAmountIn": "500000000000", "slippageBps": 50 },
    { "maxAmountIn": "1000000000000", "slippageBps": 100 }
  ],
  "measuredAt": "2024-01-15T12:00:00Z"
}
```

- `maxAmountIn`: Maximum input amount (in smallest units) at this slippage
- `slippageBps`: 50 = 0.5%, 100 = 1.0%

### Assets Output

```json
{
  "assets": [
    { "chainId": "1", "assetId": "0xa0b86...", "symbol": "USDC", "decimals": 6 },
    { "chainId": "137", "assetId": "0x2791...", "symbol": "USDC", "decimals": 6 }
  ],
  "measuredAt": "2024-01-15T12:00:00Z"
}
```

- List of all tokens supported by Stargate
- Mapped from `chainKey` to `chainId`

## 🎯 Performance Tips

### 1. Parallel Routes

The service processes routes sequentially. For many routes, consider batching:

```typescript
// Process in batches of 5
const batchSize = 5;
for (let i = 0; i < routes.length; i += batchSize) {
  const batch = routes.slice(i, i + batchSize);
  await service.getSnapshot({ routes: batch, notionals, includeWindows });
}
```

### 2. Cache Assets

Assets rarely change. Cache the result:

```typescript
let cachedAssets = null;
let cacheTime = 0;
const CACHE_TTL = 3600000; // 1 hour

async function getAssets() {
  if (cachedAssets && Date.now() - cacheTime < CACHE_TTL) {
    return cachedAssets;
  }
  
  const snapshot = await service.getSnapshot({...});
  cachedAssets = snapshot.listedAssets;
  cacheTime = Date.now();
  return cachedAssets;
}
```

### 3. Optimize Liquidity Search

For faster liquidity depth (fewer API calls), reduce binary search iterations:

```typescript
// In src/services/index.ts
for (let i = 0; i < 16; i++) {  // Reduced from 24
  // ... binary search logic
}
```

Trade-off: Less accurate threshold detection.

## 🔧 Integration with Web UI

The repo includes a web UI at `apps/web`. To test your plugin:

### 1. Update Plugin Registry

Edit `apps/web/src/config.ts`:

```typescript
const PLUGIN_REGISTRY = {
  '@every-plugin/layerzero': {
    remoteUrl: 'http://localhost:3000/remoteEntry.js',
    version: '1.0.0',
    description: 'LayerZero/Stargate data provider',
  }
};
```

### 2. Start Development Servers

```bash
# Terminal 1: Start plugin dev server
cd packages/layerzero-plugin
bun run dev

# Terminal 2: Start web UI
cd apps/web
bun run dev
```

### 3. Open Browser

Navigate to `http://localhost:3001` and test the plugin through the UI.

## 📝 Next Steps

1. **Test with Real Data**: Run against live APIs
2. **Verify Metrics**: Check that all 4 metrics return valid data
3. **Performance Test**: Measure latency and throughput
4. **Error Handling**: Test with invalid routes/tokens
5. **Documentation**: Add any provider-specific notes

## 🎉 You're Ready!

The LayerZero/Stargate plugin is fully implemented and ready to use. Check the main README.md for detailed documentation.

### Quick Links

- [README.md](./README.md) - Full documentation
- [IMPLEMENTATION_SUMMARY.md](./IMPLEMENTATION_SUMMARY.md) - Implementation details
- [src/](./src/) - Source code
- [__tests__/](./src/__tests__/) - Tests

### Support

For issues or questions:
1. Check the README.md for API documentation
2. Review IMPLEMENTATION_SUMMARY.md for architecture details
3. Inspect console logs for debugging info
4. Verify API endpoints are accessible
