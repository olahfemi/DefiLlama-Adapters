# DefiLlama Adapters - AI Coding Agent Instructions

## Project Overview

DefiLlama Adapters is a specialized codebase for DeFi protocol TVL (Total Value Locked) and liquidation tracking. The repository contains two primary adapter types:

- **TVL Adapters** (`projects/`): Calculate protocol TVL across multiple blockchains
- **Liquidation Adapters** (`liquidations/`): Monitor liquidatable positions on lending protocols

## Core Architecture Patterns

### TVL Adapter Structure
Every TVL adapter exports an object with chain-specific functions:
```javascript
module.exports = {
  ethereum: { tvl, borrowed, pool2, staking },
  polygon: { tvl },
  methodology: "Description of TVL calculation approach"
}
```

Key conventions:
- Use `projects/helper/` utilities extensively - never add external npm packages
- `api` parameter provides chain context: `api.chain`, `api.block`, `api.timestamp`
- Call `sumTokens2()` to aggregate token balances instead of manual calculations
- Transform addresses using `getChainTransform(chain)` for cross-chain compatibility

### Liquidation Adapter Structure
Liquidation adapters return liquidatable position data:
```typescript
module.exports = {
  [chain]: {
    liquidations: async () => Promise<Liq[]>
  }
}

interface Liq {
  owner: string;           // Position owner address
  liqPrice: number;        // Liquidation price in USD
  collateral: string;      // Token address with chain prefix
  collateralAmount: string; // Raw token amount (not decimal-adjusted)
}
```

## Essential Helper Functions

### Core Imports
```javascript
const { sumTokens2 } = require('../helper/unwrapLPs');
const { getChainTransform } = require('../helper/portedTokens');
const ADDRESSES = require('../helper/coreAssets.json');
```

### Common Patterns
- **Multi-chain helpers**: Use `aaveV3Export()`, `aaveV2Export()` for protocol variants
- **Token aggregation**: `sumTokens2({ api, tokensAndOwners })` for balance calculations  
- **LP unwrapping**: Set `resolveLP: true` to unwrap liquidity positions
- **Address transformation**: Always transform addresses for proper token identification
- **Blacklist filtering**: Use `blacklistedTokens` array to exclude problematic tokens

## Testing & Development Workflow

### Testing Adapters
```bash
# Test TVL adapter
node test.js projects/protocol/index.js

# Test with historical data
node test.js projects/aave-v3/index.js 2024-10-16

# Test liquidation adapter
npx ts-node ./liquidations/test.ts ./liquidations/protocol/index.ts
```

### Environment Setup
- Use `.env` file to override RPC endpoints: `ETHEREUM_RPC="..."`
- Chain names follow format in `projects/helper/chains.json`
- SDK updates: `npm update @defillama/sdk` for latest features

## Critical Constraints

### Adapter Rules
- **No external dependencies**: Only use existing helper functions and SDK
- **Chain validation**: Ensure chain names exist in `projects/helper/chains.json`
- **Export key validation**: Only use whitelisted keys from `projects/helper/whitelistedExportKeys.json`
- **Performance**: Liquidation adapters must complete within 15 minutes (AWS Lambda limit)

### File Organization
```
projects/
  protocol-name/
    index.js       # Main adapter or multi-version wrapper
    v1.js, v2.js   # Version-specific implementations
  helper/          # Shared utilities (DO NOT MODIFY without approval)

liquidations/
  protocol-name/
    index.ts       # TypeScript implementation
```

## Advanced Patterns

### Protocol Factory Pattern
For protocols with multiple contracts, use helper factories:
```javascript
const { aaveV3Export } = require("../helper/aave");
module.exports = aaveV3Export({
  ethereum: ['0x...poolAddress'],
  polygon: ['0x...poolAddress']
});
```

### Cross-chain Token Mapping
```javascript
const transformAddress = getChainTransform(chain);
// Automatically handles cross-chain token address standardization
```

### Historical Hallmarks
Add significant events affecting TVL:
```javascript
module.exports.hallmarks = [
  [1659630089, "Start OP Rewards"],
  [1650471689, "Start AVAX Rewards"]
];
```

## Data Sources & Integration

### Subgraph Integration
```javascript
const { request, gql } = require('graphql-request');
const query = gql`query { ... }`;
const data = await request(subgraphUrl, query);
```

### RPC Optimization
- Use `api.multiCall()` for batch contract calls
- Leverage `projects/helper/cache/` for persistent data
- Use `getConfig()` for external API caching

## Common Pitfalls to Avoid

1. **Address Format**: Always include chain prefix for token addresses (`ethereum:0x...`)
2. **Decimal Handling**: Use raw token amounts, let helpers handle decimal conversion
3. **Export Structure**: Don't export TVL functions at root level - nest under chain names
4. **Testing**: Always test with historical timestamps to verify accuracy
5. **Performance**: Batch RPC calls and use caching for expensive operations

Remember: DefiLlama prioritizes accuracy and transparency. When in doubt, use existing helper patterns rather than custom implementations.