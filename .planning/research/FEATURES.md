# Feature Research

**Domain:** On-chain synthetic derivatives engine (Injective / CosmWasm)
**Researched:** 2026-02-16
**Confidence:** MEDIUM-HIGH (verified against competitor docs, Injective docs, and protocol architectures; some Injective-specific details are LOW confidence due to sparse CosmWasm oracle integration docs)

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete or unsafe.

#### Oracle / Price Feed

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| External price feed integration | No price = no product. Every synth engine resolves PnL against an external oracle price. Synthetix uses Pyth, GMX uses Chainlink Data Streams, dYdX uses its own oracle network. | MEDIUM | Injective provides native oracle module with Band, Pyth, Coinbase providers. CosmWasm contracts query via `InjectiveQuerier` using `OraclePriceResponse`. The `injective-price-oracle` relayer handles off-chain price submission. |
| Price staleness rejection | Stale prices enable exploits (arbitrage against known-outdated prices, unfair liquidations). Every production protocol checks price freshness. Synthetix rejects settlement on stale oracles; GMX uses min/max price spread. | LOW | Store `last_updated_at` timestamp with each price update. Reject position opens/closes/liquidations when `block_time - last_updated_at > MAX_STALENESS` (e.g., 60-120s). Simple guard, critical safety. |
| Price deviation guard | Prevents relayer from posting wildly incorrect prices (compromised feed, API error). GMX caps price impact at 50-1000 bps. Standard practice. | LOW-MEDIUM | Compare new price against last stored price. Reject if deviation exceeds threshold (e.g., 10% per update). Requires tuning per market volatility. |

#### Position Management

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Open long/short position | Core product. Every perp/synth platform supports directional exposure. User deposits collateral, specifies side, size, leverage. | MEDIUM | Position struct: `{owner, side, size, entry_price, collateral, leverage, opened_at}`. Validate collateral >= size/leverage. Emit events. |
| Close position (full) | Users must be able to exit. GMX, Synthetix, dYdX, Hyperliquid all support full close with PnL settlement. | LOW-MEDIUM | Calculate PnL at current oracle price, return collateral +/- PnL to user. Must handle case where PnL loss exceeds collateral (bad debt). |
| PnL calculation | Real-time unrealized PnL is fundamental to any derivatives UI/integration. Every protocol exposes this. | LOW | `long_pnl = (current - entry) * size`, `short_pnl = (entry - current) * size`. Query-only, no state change. |
| Collateral deposit (INJ native) | Users need to fund positions. INJ native simplifies to `info.funds` validation in CosmWasm. | LOW | Validate sent funds match declared collateral. No CW20 token handling needed for MVP. |

#### Risk Management

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Liquidation mechanism (MMR-based) | Without liquidation, the protocol takes unbounded losses. Every protocol has this: Synthetix (gradual, MEV-resistant), GMX (threshold at 0.4-1% of position size), Hyperliquid (partial then full), dYdX (automatic). | HIGH | When `collateral + pnl <= MMR_threshold`, anyone can call `liquidate(position_id)`. Must calculate correct liquidation price, handle fee distribution, prevent reentrancy. Most complex table-stakes feature. |
| Liquidation incentive (keeper reward) | Without incentive, nobody liquidates. Every protocol pays liquidators: GMX pays from remaining collateral, dYdX uses insurance fund, Hyperliquid uses on-chain liquidation bots. | LOW | Pay `liquidation_fee` (percentage of remaining collateral) to caller. Simple transfer after liquidation succeeds. |
| Maximum leverage cap | Prevents extreme risk. GMX dynamically caps leverage based on OI. Hyperliquid enforces per-market. dYdX caps at 25x. Inlet specifies 10x. | LOW | Reject `open_position` when `leverage > MAX_LEVERAGE`. Config per market. |
| Minimum collateral requirement | Prevents dust positions that cost more gas to liquidate than they're worth. Standard across protocols. | LOW | Reject positions below `MIN_COLLATERAL` (e.g., 0.1 INJ). |

#### Multi-Market

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Factory pattern (per-market contracts) | Enables BTC/USD, ETH/USD, etc. without monolithic contract. GMX V2 uses isolated markets. Perennial uses market-isolated derivatives. Inlet's design specifies this. | MEDIUM | Factory contract stores market code_id, instantiates per pair. Uses `instantiate2` for predictable addresses. Markets are isolated: one market's bad debt doesn't affect others. |
| Market configuration (per-market params) | Each market has different volatility, appropriate leverage, MMR. GMX V2 configures per-pool. Standard practice. | LOW | `MarketConfig { max_leverage, mmr_ratio, min_collateral, oracle_source, liquidation_fee_pct }` stored per market contract. |

### Differentiators (Competitive Advantage)

Features that set Inlet apart. Not required for launch but create meaningful value.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Injective native oracle module integration | Unlike generic CosmWasm chains, Injective has a built-in oracle module queryable from contracts via `InjectiveQuerier`. This means Inlet can read Band/Pyth/Coinbase prices natively without deploying a separate oracle contract. Reduces trust assumptions vs. custom oracle contract. | MEDIUM | Uses `injective-cosmwasm` crate's `OraclePriceResponse` and `create_oracle_query_handler()`. Can fall back to relayer-pushed prices when native oracle is unavailable for a pair. Unique to Injective -- not possible on generic CosmWasm chains. |
| Injective auto-execution (BeginBlocker) | Injective can auto-execute CosmWasm contracts every block without external keepers. Could enable automatic price updates, liquidation sweeps, or funding rate application without relayer dependency. No other CosmWasm chain offers this. | HIGH | Requires careful gas management. Could eliminate relayer for certain operations. Significant differentiator but complex to implement safely. Defer to post-MVP. |
| Partial position close | GMX and Hyperliquid support partial closes. Many simpler protocols only support full close. Partial close allows traders to take profit on a portion while maintaining exposure. | MEDIUM | Close N% of position: reduce size proportionally, settle proportional PnL, keep remaining position. Must recalculate effective entry price or track realized vs unrealized PnL. |
| Collateral adjustment (add/remove) | GMX allows deposit/withdraw collateral on open positions to adjust leverage and liquidation price. Adds flexibility without opening new positions. | MEDIUM | Add collateral: increase collateral, decrease effective leverage. Remove collateral: must validate remaining margin > MMR. Requires recalculating liquidation price. |
| Multiple positions per market per user | Inlet's design allows multiple positions per user per market (unlike some protocols that merge into a single net position). Enables separate strategies: one high-leverage scalp + one low-leverage hold. | LOW-MEDIUM | Use position_id indexing rather than (user, market) as unique key. More state to manage but straightforward. Matches Inlet PROJECT.md spec. |
| Insurance fund | GMX, dYdX, Synthetix all maintain insurance funds to cover bad debt when liquidations don't fully cover losses. Without it, the protocol itself becomes insolvent. | MEDIUM | Collect portion of fees into insurance fund. Draw from fund when `collateral + pnl < 0` at liquidation. Critical for protocol solvency but can start with a simple balance in factory contract. |
| TWAP price for liquidation | Perpetual Protocol uses 7-minute TWAP of index price for liquidation conditions. Protects against flash-crash liquidations from momentary price wicks. | MEDIUM-HIGH | Requires storing price history (ring buffer of recent prices). Calculate time-weighted average over configurable window. More complex oracle integration but meaningful safety improvement. |
| Configurable oracle source per market | Different markets may need different oracle providers (Band for some pairs, Pyth for others, Coinbase for majors). Injective supports all three natively. | LOW-MEDIUM | `MarketConfig.oracle_type` enum with `{Band, Pyth, Coinbase, PriceFeeder}`. Each market queries the appropriate oracle type via `InjectiveQuerier`. |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem good but create problems for Inlet's scope and design philosophy.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Orderbook / matching engine | "Real" exchange feel, limit orders, order types | Inlet is explicitly NOT an orderbook DEX. Injective already has a native orderbook module. Building a second one in CosmWasm would be redundant, complex, and inferior to the native module. Contradicts core design. | Use Injective's native exchange module for orderbook trading. Inlet provides pure synthetic exposure against oracle price -- different product, different use case. |
| Funding rate mechanism | Perp exchanges use funding to converge mark price to index. Hyperliquid charges hourly, dYdX every 8 hours. Expected in "real" perps. | Inlet is NOT a perpetual futures exchange -- it's a synthetic exposure engine. There's no mark price separate from oracle price (positions settle directly against oracle). Funding rates solve a problem Inlet doesn't have. PROJECT.md stubs this to 0. | Keep funding rate at 0 for MVP. If mark/oracle divergence becomes an issue (it shouldn't with direct oracle settlement), reconsider post-validation. |
| Cross-margin across markets | Synthetix V3 and Hyperliquid use cross-margin where collateral is shared across positions in different markets. Capital efficient. | Enormously complex. Requires global portfolio-level margin calculation across all market contracts. Factory pattern with isolated markets directly conflicts with cross-margin. Every protocol that ships cross-margin (Synthetix, Hyperliquid) took years. | Isolated margin per market is the correct MVP approach. GMX V2 also uses isolated markets. Cross-margin is a v3+ feature if ever. |
| LP / liquidity provision | GMX and Perennial rely on LPs as counterparty. Synthetix requires SNX stakers. | Inlet's design is vault/protocol-as-counterparty or direct collateral model -- no external LP pool needed. Adding LP mechanics means building a second product (liquidity management) on top of the derivatives engine. | Protocol or vault absorbs counterparty risk. Insurance fund covers bad debt. Simpler, faster to ship, validates core hypothesis. |
| Multi-collateral (CW20 tokens, stablecoins) | "Let me post USDT as collateral." Capital flexibility. | CW20 token handling adds complexity (allowances, transfer hooks, decimal normalization). INJ-only simplifies every collateral flow to native coin validation. MVP constraint per PROJECT.md. | INJ-only for MVP. Add USDC/USDT collateral in v2 if validated. |
| Advanced order types (limit, stop-loss, take-profit, TWAP) | Traders expect conditional orders from centralized exchange experience. GMX supports limit, stop, TP/SL, TWAP. | Each order type requires a separate execution path, keeper infrastructure to trigger at price, and gas-efficient storage of pending orders. Massive surface area expansion. Inlet is testnet prototype -- clarity over completeness. | Support market orders only (immediate execution at oracle price). Advanced orders are a v2 feature that can be layered on top. |
| Auto-deleveraging (ADL) | GMX uses ADL to close profitable positions when market insolvency risk is high. Last-resort mechanism. | ADL is the nuclear option -- forcibly closing profitable users' positions. Complex queue mechanics (rank by profit, leverage, size). Only needed when insurance fund is depleted and protocol faces insolvency. Extremely unlikely in testnet scale. | Insurance fund as primary bad-debt backstop. If insurance fund is depleted, pause market (admin action). ADL is a mainnet-scale feature. |
| Frontend UI | Users want a web interface. | Inlet scope is contracts + relayer. Building a frontend is a separate project. PROJECT.md explicitly excludes it. | CLI scripts or integration tests demonstrate functionality. Frontend is a separate milestone after contract validation. |
| Portfolio margin | Hyperliquid's portfolio margin margins all positions + spot together. Maximum capital efficiency. | Requires unified account model across all markets, real-time portfolio-level risk calculation, and cross-contract state. Fundamentally incompatible with isolated factory pattern. | Isolated margin per position. Users manage risk manually across markets. |

## Feature Dependencies

```
[Oracle Price Feed Integration]
    |
    +--requires--> [Price Staleness Check]
    |                  |
    |                  +--enhances--> [Price Deviation Guard]
    |
    +--enables--> [Open Position]
    |                 |
    |                 +--requires--> [PnL Calculation]
    |                 |                  |
    |                 |                  +--enables--> [Close Position (Full)]
    |                 |                  |                 |
    |                 |                  |                 +--enhances--> [Partial Close]
    |                 |                  |
    |                 |                  +--enables--> [Liquidation Mechanism]
    |                 |                                    |
    |                 |                                    +--requires--> [Liquidation Incentive]
    |                 |                                    |
    |                 |                                    +--enhances--> [Insurance Fund]
    |                 |                                    |
    |                 |                                    +--enhances--> [TWAP for Liquidation]
    |                 |
    |                 +--enhances--> [Collateral Adjustment]
    |                 |
    |                 +--enhances--> [Multiple Positions per User]
    |
    +--enables--> [Factory Pattern]
                      |
                      +--requires--> [Market Configuration]
                      |
                      +--enhances--> [Configurable Oracle Source]

[Injective Native Oracle] --enhances--> [Oracle Price Feed Integration]

[Injective Auto-Execution] --enhances--> [Liquidation Mechanism]
                           --enhances--> [Oracle Price Feed Integration]
```

### Dependency Notes

- **Oracle Price Feed requires Price Staleness Check:** Without staleness checks, the oracle feed is a liability. A stale price can be exploited for arbitrage or unfair liquidations. These must ship together.
- **Open Position requires PnL Calculation:** PnL must be computable at open time (even if zero) to validate margin. Close and liquidation both depend on PnL.
- **Liquidation requires Liquidation Incentive:** A liquidation mechanism without keeper reward is dead code. No one will call it.
- **Factory Pattern requires Market Configuration:** Each market needs its own parameters. Without per-market config, the factory has no value over a monolithic contract.
- **Insurance Fund enhances Liquidation:** Without insurance fund, bad debt (when loss > collateral) has no backstop. Protocol absorbs loss implicitly. Insurance fund makes this explicit and bounded.
- **TWAP enhances Liquidation:** TWAP protects against wick liquidations but is not required for basic functionality. Can be added after core liquidation works.
- **Partial Close enhances Close Position:** Full close must work first. Partial close is a proportional reduction of the same logic.
- **Injective Auto-Execution enhances everything:** Block-level execution can replace relayer for price updates and enable automatic liquidation sweeps, but adds complexity and is not required when a relayer exists.

## MVP Definition

### Launch With (v1)

Minimum viable product -- what's needed to validate the synthetic exposure concept on Injective testnet.

- [ ] **Oracle price feed via relayer** -- Core dependency. TypeScript relayer pushes prices to market contract. Without price, nothing works.
- [ ] **Price staleness rejection** -- Must ship with oracle. Reject actions when price is stale (>60s). Simple timestamp check.
- [ ] **Price deviation guard** -- Must ship with oracle. Reject price updates deviating >10% from last known price. Simple comparison.
- [ ] **Factory contract** -- Creates and registers market contracts. Stores market code_id. Uses `instantiate2` for predictable addresses.
- [ ] **Market contract with position open (long/short)** -- Core product. Validate collateral, leverage <= 10x, min collateral. Store position.
- [ ] **PnL calculation (query)** -- Users must see unrealized PnL. Pure query, no state change.
- [ ] **Full position close** -- Exit with PnL settlement. Return collateral +/- PnL.
- [ ] **MMR-based liquidation with keeper reward** -- Safety net. Anyone can liquidate undercollateralized positions for a fee.
- [ ] **Per-market configuration** -- max_leverage, mmr_ratio, min_collateral, liquidation_fee, oracle config.
- [ ] **Integration tests (injective-test-tube)** -- Validate all flows end-to-end in simulated Injective environment.

### Add After Validation (v1.x)

Features to add once core is working and deployed on testnet.

- [ ] **Partial position close** -- Add when traders request it. Proportional PnL settlement.
- [ ] **Collateral adjustment (add/remove on open position)** -- Add when users want to adjust leverage without closing.
- [ ] **Insurance fund** -- Add when first bad-debt scenario is encountered in testing. Collect fees, backstop liquidations.
- [ ] **Injective native oracle module queries** -- Switch from relayer-only to querying Band/Pyth/Coinbase via `InjectiveQuerier`. Reduces relayer dependency.
- [ ] **Configurable oracle source per market** -- Different oracle providers per market pair.
- [ ] **Multiple positions per user per market** -- Add when single-position-per-market feels limiting. Requires position_id indexing.

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] **TWAP price for liquidation** -- Defer until wick-liquidation becomes a real problem. Requires price history storage.
- [ ] **Injective auto-execution for liquidation sweeps** -- Defer until relayer-based liquidation is proven insufficient. Complex gas management.
- [ ] **Multi-collateral (USDC, CW20)** -- Defer until INJ-only collateral is validated as limiting.
- [ ] **Advanced order types** -- Defer entirely. Market orders only for MVP/v1.x.
- [ ] **Funding rate mechanism** -- Defer unless oracle/mark price divergence becomes a measurable problem.
- [ ] **ADL (auto-deleveraging)** -- Defer until mainnet scale where insurance fund depletion is possible.
- [ ] **Cross-margin** -- Defer indefinitely. Requires architectural rethink.

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Oracle price feed (relayer) | HIGH | MEDIUM | P1 |
| Price staleness rejection | HIGH | LOW | P1 |
| Price deviation guard | HIGH | LOW | P1 |
| Factory contract | HIGH | MEDIUM | P1 |
| Open long/short position | HIGH | MEDIUM | P1 |
| PnL calculation | HIGH | LOW | P1 |
| Full position close | HIGH | LOW-MEDIUM | P1 |
| MMR liquidation + keeper reward | HIGH | HIGH | P1 |
| Per-market configuration | MEDIUM | LOW | P1 |
| Integration tests | HIGH | MEDIUM | P1 |
| Partial close | MEDIUM | MEDIUM | P2 |
| Collateral adjustment | MEDIUM | MEDIUM | P2 |
| Insurance fund | MEDIUM | MEDIUM | P2 |
| Native oracle queries | MEDIUM | MEDIUM | P2 |
| Multiple positions per user | MEDIUM | LOW-MEDIUM | P2 |
| Configurable oracle source | LOW-MEDIUM | LOW-MEDIUM | P2 |
| TWAP liquidation price | LOW | MEDIUM-HIGH | P3 |
| Auto-execution (BeginBlocker) | LOW | HIGH | P3 |
| Multi-collateral | LOW | MEDIUM | P3 |
| Advanced order types | LOW | HIGH | P3 |
| Funding rate | LOW | MEDIUM-HIGH | P3 |

**Priority key:**
- P1: Must have for launch (testnet validation)
- P2: Should have, add after core works
- P3: Nice to have, future consideration

## Competitor Feature Analysis

| Feature | Synthetix V3 | GMX V2 | dYdX V4 | Hyperliquid | Perennial | Inlet Approach |
|---------|-------------|--------|---------|-------------|-----------|----------------|
| **Oracle** | Pyth (low-latency) | Chainlink Data Streams (min/max spread) | Custom oracle network | Custom oracle (Pyth-derived) | Oracle-agnostic | Injective native module (Band/Pyth/Coinbase) + relayer fallback |
| **Staleness check** | Yes, rejects stale oracle | Yes, via oracle spread | Yes, validator-level | Yes, per-block | Yes | Must implement: timestamp-based rejection |
| **Position open** | Via cross-margin account | Market/limit/stop orders | Orderbook matching | Orderbook matching | Intent + AMM settlement | Market order at oracle price (simplest) |
| **Position close** | Full + partial | Full + partial | Full + partial | Full + partial | Full + partial | Full (v1), partial (v1.x) |
| **Collateral adjust** | Yes (add/remove) | Yes (deposit/withdraw) | Yes | Yes | Yes | No (v1), yes (v1.x) |
| **Margin type** | Cross-margin (native) | Isolated per market | Cross-margin default + isolated | Cross + portfolio margin | Isolated per market | Isolated per market (matches GMX V2 / Perennial) |
| **Liquidation** | Gradual, MEV-resistant | Threshold-based (0.4-1%) | Automatic, insurance-backed | Partial then full, on-chain | Dynamic | MMR threshold, anyone can trigger, keeper reward |
| **Insurance fund** | SNX stakers backstop | Protocol vault (GLP/GM) | Dedicated insurance fund | Vault-based | LP pool | Simple balance (v1.x) |
| **Funding rate** | Dynamic (skew-based) | Adaptive (utilization-based) | Premium/discount (8h) | Hourly (capped 4%/hr) | Dynamic (skew + utilization) | None (stubbed to 0) -- by design |
| **ADL** | No (gradual liquidation instead) | Yes (synthetic markets) | Yes (last resort) | Yes (last resort) | No | No -- insurance fund + market pause instead |
| **Multi-market** | Multi-pool + cross-margin | Isolated GM pools | All markets on one chain | All markets on one chain | Isolated markets | Factory pattern, isolated contracts (closest to GMX V2) |
| **Order types** | Market, limit | Market, limit, stop, TP/SL, TWAP | Full orderbook | Full orderbook | Intent + AMM | Market only (v1) |
| **Leverage max** | Up to 100x | Dynamic (decreases with OI) | Up to 25x | Up to 50x | Configurable | 10x cap (conservative, validates safely) |
| **Counterparty** | SNX stakers (debt pool) | LP vault (GM tokens) | Orderbook (peer-to-peer) | Orderbook (peer-to-peer) | LP pool (Makers) | Protocol/vault as counterparty |

### Key Takeaway from Competitor Analysis

Inlet's design (oracle-based synthetic exposure, isolated markets, protocol-as-counterparty) is most similar to **GMX V2** in its isolated market structure and **Gains Network (gTrade)** in its vault-as-counterparty model. It is deliberately simpler than Synthetix V3, dYdX V4, or Hyperliquid -- all of which took years and large teams to build. The 10x leverage cap and no-funding-rate design are appropriate simplifications for a testnet prototype.

## Injective-Specific Considerations

These are features and constraints unique to building on Injective rather than generic CosmWasm or EVM:

| Consideration | Impact | Complexity | Notes |
|---------------|--------|------------|-------|
| **Native oracle module** | Can query Band/Pyth/Coinbase prices from contract via `InjectiveQuerier` without deploying oracle contracts. Reduces trust surface. | MEDIUM | Use `injective-cosmwasm` crate. `OraclePriceResponse` struct. Must understand oracle type enum and query routing. Not well-documented for CosmWasm integration -- expect friction. |
| **Auto-execution (BeginBlocker for WASM)** | Contracts can self-execute every block. Potential for automatic liquidation without keeper, automatic price refresh. Unique to Injective. | HIGH | Gas costs per block must be carefully bounded. Risk of gas exhaustion blocking other chain operations. Powerful but dangerous. Best used for lightweight operations. |
| **Native exchange module exists** | Injective already has a built-in orderbook. Inlet must NOT compete with it. Inlet's value is synthetic exposure WITHOUT orderbook matching. | N/A | Clear product differentiation. Inlet positions settle against oracle, not against counterparties. Different product for different use case. |
| **`injective-cosmwasm` / `injective-std` crate** | Provides Injective-specific bindings (oracle, exchange, auction, insurance, tokenfactory). Less mature than `cosmwasm-std`. | LOW-MEDIUM | Pin to stable version. Use oracle queries. Avoid exchange module bindings (not needed). Watch for breaking changes between versions. |
| **`injective-test-tube`** | Integration testing framework that simulates full Injective chain. Better than `cw-multi-test` for Injective-specific features. | MEDIUM | Required for testing oracle queries and Injective-specific message routing. More setup than `cw-multi-test` but much more realistic. |
| **Governance-gated oracle privileges** | On mainnet, relayers need governance approval to push prices. On testnet, this may be simpler. | LOW (testnet) | Testnet may have different privilege requirements. Verify during deployment. Mainnet would require governance proposal. |
| **Gas model** | Injective has zero gas fees for users on native exchange module, but CosmWasm contracts still have gas costs. | LOW | Standard CosmWasm gas model applies. Optimize storage reads/writes. Position iteration in liquidation sweeps must be bounded. |

## Sources

### Protocol Documentation (MEDIUM-HIGH confidence)
- [Synthetix Perps V3 Features Explainer](https://blog.synthetix.io/perps-v3-features-release-explainer/)
- [GMX V2 Trading Docs](https://docs.gmx.io/docs/trading/)
- [dYdX V4 Documentation](https://docs.dydx.exchange/)
- [Hyperliquid Funding Docs](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/funding)
- [Hyperliquid Portfolio Margin](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/portfolio-margin)
- [Perennial Introduction](https://docs.perennial.finance/)
- [Kwenta V3 Perps Docs](https://docs.v3.kwenta.io/v3-perps/perpetual-futures)

### Injective-Specific (MEDIUM confidence -- some docs are sparse)
- [Injective Oracle Module Docs](https://docs.injective.network/developers/modules/injective/oracle)
- [injective-cosmwasm crate (0.3.3)](https://docs.rs/crate/injective-cosmwasm/latest)
- [injective-price-oracle GitHub](https://github.com/InjectiveLabs/injective-price-oracle)
- [cw-injective GitHub](https://github.com/InjectiveLabs/cw-injective)
- [Injective CosmWasm Docs](https://docs.injective.network/developers-cosmwasm/smart-contracts/)

### Oracle Design Patterns (MEDIUM confidence)
- [Ensuring Safe Oracle Usage in DeFi](https://hackmd.io/@tapir/S17-GbR-3)
- [DeFi Price Oracles - TWAP](https://composable-security.com/blog/defi-price-oracles-all-you-should-know-about-a-twap/)
- [Oracle Manipulation Risks](https://www.certik.com/resources/blog/oracle-wars-the-rise-of-price-manipulation-attacks)

### Liquidation & Risk Management (MEDIUM confidence)
- [Understanding Liquidation on Perp DEXs (2026)](https://www.bitcoin.com/get-started/understanding-liquidation-on-perp-dex/)
- [How ADL Works on Crypto Perp Platforms](https://www.coindesk.com/markets/2025/10/11/how-adl-on-crypto-perp-trading-platforms-can-shock-and-anger-even-advanced-traders)
- [How Funding Rates Work on Perp DEXs (2026)](https://www.bitcoin.com/get-started/how-funding-rates-work-on-perp-dex/)

### Architecture (MEDIUM confidence)
- [CosmWasm Instantiate2 Algorithm](https://cosmwasm.cosmos.network/core/specification/instantiate2-algo/)
- [Injective Auto-Execution Blog](https://medium.com/@sevadechev10/injectives-automatic-smart-contracts-unlocking-innovation-in-defi-a31502e6e4fc)
- [Perpetual DEX Architecture Explained](https://www.blockchainappfactory.com/blog/perpetual-dex-architecture-explained/)

---
*Feature research for: On-chain synthetic derivatives engine (Injective / CosmWasm)*
*Researched: 2026-02-16*
