# EVM PumpFun Smart Contract ⚡

**Version 2** — High-performance implementation of pump.fun-style bonding-curve and AMM mechanics for EVM chains. Deployable on Base, Ethereum, and other EVM-compatible networks.

---

## Version history

### v1 (public)

- Bonding-curve engine (PumpFun) with create pool, buy, sell, owner withdraw.
- TokenFactory: deploy ERC20 + create pool in one tx; owner-only pool address.
- AMM: PancakeFactory, PancakePair, Pair, Router (add/remove liquidity, swap, referral fee).
- Security: ReentrancyGuard, CEI in sell, owner set in constructor, excess ETH refund on createPool.
- Base + Base Sepolia in Hardhat config; optimizer + viaIR.

### v2 (private)

| Area | Improvement |
|------|-------------|
| **PumpFun** | View helpers: `getBuyQuote(token, tokenAmount)`, `getSellQuote(token, tokenAmount)`, `getTokenAmountForEth(token, ethAmount)` for frontends. Getters: `getOwner()`, `getFeeRecipient()`, `getFeeBasisPoint()`, `getMcapLimit()`, `isPoolComplete(token)`. Optional: `pause` / `unpause` for emergency halt. |
| **Router** | Allow `referree == address(0)` so swaps work without a referrer. Add `deadline` to swap functions to limit front-run risk. Admin: `setReferralFee` (and optionally owner). |
| **Pair** | Restrict `mint`, `burn`, `swap` to Router (or single core) so only the protocol can update reserves. |
| **TokenFactory** | `getTokenCount()`; `transferOwnership(address)` or `setOwner` for ownership transfer. |
| **Router** | `getAmountsIn(token, weth, amountOut)` for “how much in to get X out” quotes. |

---

## Architecture

The system has two phases per token:

1. **Bonding curve (PumpFun)** — New tokens trade on a constant-product style curve until a threshold (mcap or sell-down) is hit.
2. **Graduation** — Completed curves can be withdrawn by the protocol owner; liquidity can be moved to a Uniswap-style AMM (Router + Factory/Pair) for continued trading.

```
┌─────────────────┐      ┌───────────────────┐     ┌─────────────────┐
│   TokenFactory  │────> │      PumpFun      │────>│ PancakeFactory  │
│ (deploy ERC20 + │      │  (bonding curve   │     │ + Router + Pair │
│  create pool)   │      │ buy/sell/withdraw)│     │ (AMM post-grad) │
└─────────────────┘      └───────────────────┘     └─────────────────┘
```

---

## Contract Overview

| Contract | Role |
|----------|------|
| **PumpFun** | Bonding-curve: `createPool`, `buy`, `sell`, `withdraw` (owner). Quotes: `getBuyQuote`, `getSellQuote`, `getTokenAmountForEth`. Getters: `getOwner`, `getFeeRecipient`, `getFeeBasisPoint`, `getMcapLimit`, `getTokenTotalSupply`, `getInitialVirtualReserves`, `isPoolComplete`. Emergency: `pause` / `unpause`. |
| **TokenFactory** | Deploys ERC20 + `createPool` in one tx. Owner-only `setPoolAddress`. v2: `getTokenCount()`, `transferOwnership(address)`. |
| **Token** | Standard ERC20 (OpenZeppelin); minted to creator with fixed total supply. |
| **PancakeFactory** | UniswapV2-style factory: `createPair(tokenA, tokenB)` via CREATE2, `feeTo`, `txFee`, `getPair`. Implements `IFactory`. |
| **PancakePair** | Minimal pair: `initialize(token0, token1)`; holds pair state for Router. |
| **Pair** | Full AMM pair: `mint`/`burn`/`swap` restricted to `core` (set by factory via `setCore`). `approval`, `transferETH`, `getReserves`, `kLast`, `core()`. |
| **Router** | Liquidity: `addLiquidityETH`, `removeLiquidityETH`. Swaps: `swapTokensForETH(..., referree, deadline)`, `swapETHForTokens(..., referree, deadline)`; referree may be `address(0)`. Quotes: `getAmountsOut`, `getAmountsIn`. Owner-only `setReferralFee`. |
| **Factory.sol** | Interface `IFactory`: `getPair`, `feeTo`, `txFee`. |

---

## Bonding Curve (PumpFun)

- **Pricing**: Constant-product over virtual reserves.  
  `ethCost = (virtualEthReserves * virtualTokenReserves) / (virtualTokenReserves - tokenAmount) - virtualEthReserves`
- **Reserves**: Per-token curve state: `virtualTokenReserves`, `virtualEthReserves`, `realTokenReserves`, `realEthReserves`, `tokenTotalSupply`, `mcapLimit`, `complete`.
- **Completion**: Curve marks `complete` when mcap exceeds `mcapLimit` or real token reserve percentage falls below 20%. Only then can owner call `withdraw(token)` to pull real ETH and tokens.
- **Fees**: Create fee (flat ETH) and trading fee (basis points) sent to `feeRecipient`. Excess ETH in `createPool` is refunded to caller.
- **Security**: `ReentrancyGuard` on pool creation and trading; CEI in `sell`; owner set in constructor; no external calls before state updates in critical paths. v2: `pause`/`unpause` for emergency halt.

---

## Tech Stack

| Component | Version / choice |
|-----------|-------------------|
| Solidity | `0.8.24` |
| Compiler | Optimizer enabled, `runs: 200`, `viaIR: true` (resolves stack-too-deep in Router) |
| Framework | Hardhat 2.x + Ignition |
| Dependencies | OpenZeppelin Contracts v5, ethers v6, TypeScript |
| Networks | Base (8453), Base Sepolia (84532); configurable via `hardhat.config.ts` |

---

## Development

### Prerequisites

- Node.js ≥ 18
- npm or yarn

### Install

```bash
npm install --legacy-peer-deps
```

### Build

```bash
npx hardhat compile
```

### Test

```bash
npx hardhat test
```

Tests cover TokenFactory deployment, PumpFun pool creation, and buy/sell on the bonding curve (see `test/test.ts`).

### Deploy (Base)

Set deployer key (never commit):

```bash
# Windows (PowerShell)
$env:PRIVATE_KEY = "0x..."

# Linux/macOS
export PRIVATE_KEY=0x...
```

Deploy with Hardhat Ignition (example for Lock module):

```bash
# Base Mainnet
npx hardhat ignition deploy ignition/modules/Lock.ts --network base

# Base Sepolia
npx hardhat ignition deploy ignition/modules/Lock.ts --network baseSepolia
```

Add and run Ignition modules for PumpFun, TokenFactory, and (optionally) PancakeFactory/Router as needed.

### Verify on Explorer

Configure Etherscan/BaseScan API key in `hardhat.config.ts` and use:

```bash
npx hardhat verify --network base <DEPLOYED_ADDRESS> <CONSTRUCTOR_ARGS>
```

---

## Configuration

- **PumpFun constructor**: `feeRecipient`, `createFee` (wei), `feeBasisPoint` (e.g. 100 = 1%).
- **TokenFactory**: Owner must call `setPoolAddress(pumpFunAddress)` after deployment. Use `transferOwnership` to change owner.
- **PancakeFactory**: `feeToSetter` in constructor; then `setFeeTo`, `setTxFee`, `setFeeToSetter` as needed for Router compatibility.
- **Pair (v2)**: After the factory creates a pair, it must call `pair.setCore(routerAddress)` once so the Router can call `mint`/`burn`/`swap`. Only the factory can call `setCore`, and only before a core is set.
- **Router**: Swaps require a `deadline` (e.g. `block.timestamp + 300`). Use `address(0)` as `referree` when no referrer.

---

## License

MIT — see [LICENSE](LICENSE).

---

## Contact

- [Telegram](https://t.me/trustdev_eth)
- [Twitter](https://x.com/trustdev_eth)
