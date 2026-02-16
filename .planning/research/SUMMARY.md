# Project Research Summary

**Project:** Inlet - Synthetic Derivatives Engine
**Domain:** On-chain DeFi derivatives (CosmWasm on Injective)
**Researched:** 2026-02-16
**Confidence:** MEDIUM-HIGH

## Executive Summary

Inlet is a synthetic derivatives engine on Injective that provides leveraged long/short exposure to crypto assets (BTC/USD, ETH/USD, etc.) settled against oracle prices. The research confirms this is a well-understood product category with clear architecture patterns from established protocols like GMX V2, Synthetix, and Hyperliquid. The recommended approach is a factory pattern with isolated market contracts, oracle-based settlement, and protocol-as-counterparty, deliberately simplified from cross-margin perpetuals to validate core mechanics on testnet.

The Injective stack is stable and production-ready: injective-std 1.14.1 with cosmwasm-std 2.1 provides native oracle module integration unavailable on generic CosmWasm chains. The critical dependencies are oracle price feeds (via injective-price-oracle relayer), injective-test-tube for realistic integration testing, and injective-math for fixed-point arithmetic. The ecosystem is in active transition toward cosmwasm-std 3.0 (via injective-std 1.18.x-beta), but the stable 2.x line is the correct choice for near-term testnet deployment.

The primary risks are oracle-related: stale prices, price deviation exploits, and relayer single-point-of-failure. These must be addressed in Phase 1 with staleness checks, deviation guards, and circuit breakers. Secondary risks include precision handling (INJ has 18 decimals), underwater positions creating bad debt without recovery mechanisms, and gas-griefing via unbounded position iteration. Insurance fund coverage and position limits mitigate these. The project scope correctly defers advanced features (funding rates, cross-margin, ADL) to focus on core validation.

## Key Findings

### Recommended Stack

Inlet should use **injective-std 1.14.1 (stable)** with **cosmwasm-std 2.1** for on-chain contracts, **@injectivelabs/sdk-ts 1.16.12** for the TypeScript relayer, and **injective-test-tube 1.14.0-rc2** for integration testing. These versions form a compatible set targeting Injective testnet (injective-888) running chain v1.17.x.

**Core technologies:**
- **injective-std 1.14.1:** Protobuf-based Injective chain bindings. Replaces deprecated injective-cosmwasm. Provides OracleQuerier for native oracle module, critical for price feeds without deploying oracle contracts.
- **injective-math 0.3.0:** Fixed-point FPDecimal library. Essential for DeFi math. Prevents floating-point non-determinism that would cause consensus failures. Use for all margin, PnL, and liquidation calculations.
- **injective-test-tube 1.14.0-rc2:** Integration testing against compiled Injective chain Go modules. Unlike cw-multi-test, this tests real oracle module behavior, Exchange module, and WasmX features. Required for validating oracle queries.
- **@injectivelabs/sdk-ts 1.16.12:** TypeScript SDK for off-chain relayer and liquidation bot. Provides transaction construction, gRPC streaming, and message types for all Injective modules. Actively maintained, published Feb 2026.
- **cosmwasm/optimizer 0.17.0:** Production Wasm compilation. Produces deterministic, size-optimized artifacts. Replaces deprecated workspace-optimizer and rust-optimizer images.

**Critical version compatibility:** Pin injective-std and injective-test-tube to the same major.minor (both 1.14.x). They track the same injective-core version. Do NOT use cosmwasm-std 3.0 yet; it's only in injective-std beta (1.18.x). Stick with cosmwasm-std 2.1 for stability.

**Avoid:** injective-cosmwasm (deprecated, JSON messages), cosmwasm-std 3.0 (no stable injective-std support), floating-point math (non-deterministic), manual cargo build --release (non-deterministic).

### Expected Features

Based on competitor analysis (GMX V2, Synthetix V3, dYdX V4, Hyperliquid, Perennial), the feature landscape is well-defined.

**Must have (table stakes):**
- **Oracle price feed with staleness rejection:** Every protocol checks price freshness. 60-120s threshold. Without this, stale prices enable arbitrage exploits.
- **Open long/short positions:** Core product. User deposits INJ collateral, specifies side, size, leverage (1x-10x). Store position with entry price.
- **Full position close with PnL settlement:** Exit with profit/loss calculated against oracle price. Floor payout at zero (bad debt case).
- **MMR-based liquidation with keeper reward:** Anyone can liquidate positions below maintenance margin. Pay liquidation fee to caller. Standard DeFi safety mechanism.
- **Factory pattern (per-market contracts):** Isolated markets for BTC/USD, ETH/USD, etc. Factory instantiates market contracts via SubMsg+Reply. Matches GMX V2 / Perennial architecture.
- **Per-market configuration:** max_leverage, mmr_ratio, liquidation_fee_rate, oracle_source. Each market has different volatility profiles.

**Should have (competitive):**
- **Injective native oracle module integration:** Unlike generic CosmWasm chains, query Band/Pyth/Coinbase prices directly via InjectiveQuerier. Reduces relayer dependency for price reads (relayer only writes).
- **Partial position close:** GMX and Hyperliquid support this. Close N% of position, settle proportional PnL, keep remainder. Add post-MVP when requested.
- **Collateral adjustment:** Add/remove collateral on open positions to adjust leverage. Requires recalculating liquidation price. GMX supports this.
- **Multiple positions per user per market:** Enables separate strategies (one high-leverage scalp, one low-leverage hold). Use position_id indexing rather than (user, market) unique key.
- **Insurance fund:** GMX, dYdX, Synthetix all maintain funds to cover bad debt. Collect portion of fees. Draw when liquidations leave negative equity. Critical for solvency but can start simple.

**Defer (v2+):**
- **Funding rate mechanism:** Inlet settles directly against oracle price (no mark/index divergence). Funding rate solves a problem Inlet doesn't have by design. Correctly stubbed to 0.
- **Cross-margin:** Requires portfolio-level calculation across all markets. Fundamentally incompatible with isolated factory pattern. Defer indefinitely.
- **Advanced order types (limit, stop, TP/SL):** Massive surface area. Each type requires keeper infrastructure. Market orders only for MVP.
- **TWAP liquidation price:** Protects against wick liquidations but adds complexity (store price history). Defer until wick-liquidation becomes a problem.
- **Auto-deleveraging (ADL):** Last-resort mechanism to close profitable positions when insurance depleted. Only needed at mainnet scale.

### Architecture Approach

The standard architecture is a **factory + isolated market contracts** pattern with **oracle module integration** for price feeds and **off-chain relayer + liquidation bot** for operations.

**Major components:**
1. **Factory Contract:** Stores market_code_id, deploys market contracts via WasmMsg::Instantiate wrapped in SubMsg. Reply handler captures new contract address. Maintains MARKETS registry (Map<String, Addr> from "BTC/USD" to market address). Admin for migrations. No direct position or oracle logic.

2. **Market Contract (one per trading pair):** Manages positions for a single pair (BTC/USD, ETH/USD, etc.). Stores positions in IndexedMap keyed by auto-incrementing u64 position_id, with MultiIndex on owner for efficient user queries. Queries oracle module via OracleQuerier for current price on every position operation. Holds INJ collateral in Bank module balance. Executes liquidations when margin < MMR. Isolated failure domain: one market's bad debt doesn't affect others.

3. **Oracle Module (native):** Injective's built-in module. Receives MsgRelayPriceFeedPrice from relayer, stores PriceFeedState with price + timestamp. Queryable by market contracts via Stargate/Any queries using injective-std OracleQuerier. Requires governance-granted price feeder privilege (simplified on testnet).

4. **Price Relayer (TypeScript):** Fetches prices from external APIs (Binance, CoinGecko), relays to oracle module every 10s. Uses @injectivelabs/sdk-ts for transaction construction. Single point of failure: requires health monitoring, redundancy for production. Based on injective-price-oracle Go implementation (TOML dynamic feeds).

5. **Liquidation Bot (TypeScript):** Off-chain monitoring of positions. Queries positions by user or scans market state. Identifies underwater positions (collateral + pnl <= MMR threshold). Calls ExecuteMsg::Liquidate for keeper reward. No privileged access required (permissionless).

**Key patterns:**
- **Factory SubMsg+Reply:** Factory sends WasmMsg::Instantiate as SubMsg with reply_on_success. Reply captures new contract address from instantiation response. Standard CosmWasm factory pattern.
- **Oracle queries via injective-std:** Use OracleQuerier::oracle_price() with OracleType::PriceFeed. Returns PricePairState with price string + timestamp. Parse to Decimal256, validate staleness.
- **IndexedMap for positions:** Primary key is u64 position_id (auto-increment). MultiIndex on owner (Addr) for per-user queries. Efficient O(1) lookup by ID, paginated range queries by owner.
- **Bot-callable liquidation:** ExecuteMsg::Liquidate { position_id } is permissionless. Contract validates margin < MMR, pays liquidation_fee to caller, returns remaining collateral to owner.

**Project structure:**
```
contracts/factory/      # Factory contract (deploys markets)
contracts/market/       # Market contract (positions, collateral, liquidation)
packages/inlet-types/   # Shared msg types (Position, Side, configs)
relayer/                # TypeScript relayer (price feeds, liquidation bot)
tests/                  # Integration tests (injective-test-tube)
```

### Critical Pitfalls

The top pitfalls from research with prevention strategies:

1. **Stale Oracle Price Exploitation (CRITICAL):** Relayer fails, price becomes stale. Attacker opens/closes positions at outdated price, extracting risk-free profit. Liquidations fire (or fail) on stale data.
   - **Prevention:** Store last_price_update_timestamp. Assert block_time - last_update < MAX_STALENESS (60-120s) on every position operation. Reject if stale. Add circuit breaker: reject price deviating >15% from previous update. Implement admin pause_market() function. Run multiple relayer instances for redundancy.

2. **Rounding Direction Favoring Users Over Protocol (CRITICAL):** Fixed-point math truncates. If truncation consistently favors users (collateral requirement rounds down, PnL rounds up), dust losses compound to drain protocol. Worse: liquidation checks rounding in user's favor means unliquidatable positions.
   - **Prevention:** Establish rounding policy: collateral requirements round UP, PnL payouts round DOWN, liquidation thresholds round AGAINST user. Use explicit checked_mul_ceil() / checked_div_floor() methods from cosmwasm-std. Comment every arithmetic function with rounding justification. Property test: "protocol never pays out more than received."

3. **Underwater Positions and Bad Debt Without Recovery (CRITICAL):** Position loss exceeds collateral (collateral + pnl < 0). No rational liquidator executes (they'd lose money). Position stuck, protocol insolvent.
   - **Prevention:** Always allow liquidation of underwater positions. Liquidator receives whatever collateral remains (even if less than standard fee). Track bad_debt: Uint128 in contract state. Insurance fund (Phase 2) covers bad debt. Auto-Deleveraging (Phase 3+) as last resort. Never revert liquidate() on negative equity; cap payout at zero and record shortfall.

4. **Precision Mismatch Between Decimal, Uint128, and INJ Denomination (CRITICAL):** INJ uses 18 decimals (1 INJ = 10^18 inj). CosmWasm Decimal has 18 fractional digits. Uint128 has none. Mixing without explicit conversion causes off-by-10^18 errors. Position opened with "1 INJ" actually has 10^-18 INJ (zero), or payout of "1" is actually 10^18 INJ (draining contract).
   - **Prevention:** Define PRECISION_FACTOR: Uint128 = 10^18 constant. Create typed wrappers: struct Inj(Uint128) for raw, struct Price(Decimal). Force explicit .to_raw() / .from_raw(). Validate oracle price ranges (BTC > $1000, < $10M). Name variables with units: collateral_raw (attoINJ), collateral_inj (Decimal). Sanity test: "deposit 1 INJ, open, close, withdraw ~1 INJ."

5. **Liquidation Race Condition Between Price Update and Execution (CRITICAL):** Liquidator detects liquidatable position at P1. Same block, relayer updates to P2 where position is healthy. Liquidation either executes at stale P1 (unfair) or fails (reads updated P2).
   - **Prevention:** Always read latest on-chain price at execution time. Liquidator does NOT supply price as parameter. Contract queries oracle module in liquidate() function. Accept that liquidation is "at market": if healthy at execution, tx fails (no penalty, just gas cost). Never cache prices across submessages.

6. **Gas Griefing via Unbounded Position Iteration (CRITICAL):** User opens hundreds of tiny positions. Operations iterating all positions (batch liquidation, portfolio queries) hit gas limit and revert. Makes positions impossible to liquidate, creating permanent bad debt.
   - **Prevention:** Enforce MAX_POSITIONS_PER_USER limit (10-20). Reject open_position if exceeded. Liquidation works per-position: liquidate(position_id), not per-user batch. Never iterate all positions in execute message. Queries paginate with .take(limit). Minimum position size (e.g., 0.01 INJ) prevents dust.

## Implications for Roadmap

Based on research, the dependency structure suggests a 5-phase roadmap:

### Phase 1: Foundation & Oracle Integration
**Rationale:** Oracle is the core dependency. No price = no product. Position logic requires working oracle queries. Must come first to unblock all other work.

**Delivers:**
- Workspace setup (Cargo workspace, contracts/factory, contracts/market, packages/inlet-types)
- Oracle query helper in market contract (query_oracle_price with OracleQuerier)
- Staleness guard logic (timestamp validation, reject operations when stale)
- Price deviation guard (circuit breaker rejecting >10-15% updates)
- TypeScript relayer scaffold (MsgRelayPriceFeedPrice via sdk-ts)
- Integration test: relayer pushes price, market contract queries, validates staleness

**Addresses:**
- STACK.md: injective-std oracle integration, injective-price-oracle setup
- FEATURES.md: Oracle price feed, staleness rejection, deviation guard (all table stakes)
- PITFALLS.md: P1 (stale oracle), P4 (precision mismatch), P8 (crate versioning)

**Avoids:**
- Stale price exploitation (staleness check is foundational)
- Crate version drift (pin exact versions in workspace setup)
- Precision mismatches (establish type system and constants early)

**Research needed:** Standard patterns. Oracle module integration well-documented in injective-std. Skip `/gsd:research-phase`.

---

### Phase 2: Position Engine
**Rationale:** Core business logic. Depends on Phase 1 (needs working oracle). Must come before factory (factory deploys this contract) and liquidation (liquidation operates on positions).

**Delivers:**
- Position storage (IndexedMap with MultiIndex on owner)
- OpenPosition logic (margin validation, leverage <= 10x, collateral >= MIN_COLLATERAL, oracle query for entry_price)
- ClosePosition logic (PnL calculation, BankMsg::Send payout)
- Position query endpoints (by ID, by owner with pagination)
- PnL calculation (query-only, unrealized PnL at current price)
- Integration tests: open → query PnL → close lifecycle

**Addresses:**
- FEATURES.md: Open long/short, Close position, PnL calculation, Collateral deposit (all table stakes)
- ARCHITECTURE.md: Position storage pattern, IndexedMap implementation
- PITFALLS.md: P2 (rounding direction), P6 (position limits), P10 (input validation), P12 (checked arithmetic)

**Avoids:**
- Rounding errors (establish policy: collateral rounds up, PnL rounds down)
- Gas griefing (enforce MAX_POSITIONS_PER_USER from start)
- Integer overflow (use checked_mul, checked_div, Uint256 for intermediates)

**Research needed:** Standard patterns. Similar to existing perpetual protocols (GMX, Margined). Skip `/gsd:research-phase`.

---

### Phase 3: Factory & Multi-Market
**Rationale:** Depends on Phase 2 (must have working market contract to deploy). Enables BTC/USD, ETH/USD, SOL/USD markets. Factory is admin for market migrations.

**Delivers:**
- Factory contract state (CONFIG, MARKETS registry, PENDING_MARKET)
- CreateMarket with SubMsg + Reply (WasmMsg::Instantiate, capture address in reply)
- Market registry queries (list markets, get market address by pair)
- Code ID update mechanism (admin can update market_code_id for future deployments)
- Migration flow (factory sends WasmMsg::Migrate to all markets)
- Integration tests: factory creates market, user opens position in that market

**Addresses:**
- FEATURES.md: Factory pattern, Per-market configuration (table stakes)
- ARCHITECTURE.md: Factory SubMsg+Reply pattern, code ID management, migration
- PITFALLS.md: P7 (reply handler state mutation), P9 (storage key collision), P13 (migration)

**Avoids:**
- Reply pseudo-reentrancy (follow Checks-Effects-Interactions in reply handler)
- Storage key collisions (establish naming convention, registry of keys)
- Migration data loss (include migrate() entry point from start, version structs)

**Research needed:** Standard patterns. Factory pattern is well-documented in CosmWasm. Skip `/gsd:research-phase`.

---

### Phase 4: Liquidation & Risk Management
**Rationale:** Depends on Phase 2 (operates on positions) and Phase 1 (needs oracle price manipulation in tests). Can proceed in parallel with Phase 3 if needed, but logically comes after multi-market support is proven.

**Delivers:**
- MMR calculation logic (effective_margin / notional_value vs maintenance_margin_ratio)
- Liquidate execute handler (permissionless, validates margin < MMR, closes position)
- Liquidation fee distribution (pay fee to liquidator, return remaining to owner)
- Bad debt tracking (handle collateral + pnl < 0 case, record bad_debt in state)
- TypeScript liquidation bot (monitor positions, call Liquidate on underwater positions)
- Integration tests: open position → price drops 15% → liquidate succeeds

**Addresses:**
- FEATURES.md: MMR-based liquidation, Liquidation incentive (table stakes)
- ARCHITECTURE.md: Bot-callable liquidation pattern
- PITFALLS.md: P3 (underwater positions / bad debt), P5 (liquidation race condition), P11 (access control)

**Avoids:**
- Underwater positions stuck (always-liquidatable guarantee)
- Liquidation race conditions (always read current state, never cache price)
- Access control gaps (validate relayer sender on oracle, but liquidation is permissionless)

**Research needed:** Standard patterns. Liquidation is well-understood in DeFi. Skip `/gsd:research-phase`.

---

### Phase 5: Hardening & Testnet Deployment
**Rationale:** Polish and safety. All core features work; now add edge case handling, circuit breakers, admin controls. Final validation before testnet.

**Delivers:**
- Edge case handling (zero collateral rejection, max leverage boundary, minimum size)
- Price deviation circuit breaker (reject updates >15% unless admin override)
- Admin controls (pause_market, update_config, emergency_withdraw for stuck funds)
- Error handling review (typed ContractError with thiserror, clear error messages)
- Event emission (wasm events for all state transitions: position_opened, position_closed, liquidated)
- Full integration test suite (property tests, fuzz boundaries, concurrent operations)
- Testnet deployment scripts (deploy factory, upload market code, create initial markets)
- Relayer deployment (run injective-price-oracle with testnet config)

**Addresses:**
- FEATURES.md: Price deviation guard (table stakes, but can be Phase 2 if time)
- PITFALLS.md: P14 (testnet/mainnet differences), P15 (test infrastructure gaps), P16 (circuit breaker)

**Avoids:**
- Price manipulation (deviation guard catches outliers)
- Testnet/mainnet assumptions (externalize config, document assumptions)
- Test coverage gaps (two-tier strategy: cw-multi-test + injective-test-tube)

**Research needed:** None. This is polish and deployment. Skip `/gsd:research-phase`.

---

### Phase Ordering Rationale

**Dependency-driven:**
- Phase 1 before all: Oracle is the foundational dependency. Positions, liquidations, everything needs price.
- Phase 2 before 3 & 4: Market contract must work standalone before factory deploys it. Liquidation operates on positions.
- Phase 3 can partially overlap 4: Factory and liquidation are independent features. But testing multi-market liquidation requires Phase 3 complete.
- Phase 5 last: Hardening only makes sense after happy path works.

**Risk-driven:**
- Phase 1 addresses CRITICAL pitfalls P1, P4, P8 (oracle, precision, versioning) immediately.
- Phase 2 addresses CRITICAL pitfalls P2, P6, P10, P12 (rounding, gas, validation, overflow) in core logic.
- Phase 4 addresses CRITICAL pitfalls P3, P5 (bad debt, race conditions) in liquidation.
- Phase 5 catches remaining MEDIUM/LOW risks (circuit breaker, testnet config).

**Architecture-driven:**
- Phase 1-2-3 builds the contract layer (oracle → positions → factory).
- Phase 4 builds the off-chain layer (liquidation bot).
- Phase 5 integrates everything for deployment.

### Research Flags

**Phases with standard patterns (skip research-phase):**
- **Phase 1 (Oracle):** Injective oracle module integration is well-documented in injective-std docs.rs and Injective docs. OracleQuerier API is straightforward.
- **Phase 2 (Positions):** Position management is standard DeFi logic. Similar to GMX, Margined Protocol, Perennial. IndexedMap pattern documented in cw-storage-plus.
- **Phase 3 (Factory):** SubMsg+Reply factory pattern is the canonical CosmWasm approach. Documented in CosmWasm book and multiple reference implementations.
- **Phase 4 (Liquidation):** Liquidation logic is well-understood across DeFi. Similar to Aave, Compound, Margined Protocol. No novel integration challenges.
- **Phase 5 (Hardening):** Polish phase. No research needed.

**No phases require `/gsd:research-phase` during planning.** All patterns are established. Research completed here is sufficient.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | injective-std 1.14.1 stable on crates.io, extensively documented. @injectivelabs/sdk-ts actively maintained (published Feb 2026). Version compatibility matrix verified across docs.rs, GitHub repos, and npm. Testnet (injective-888) runs chain v1.17.x confirmed. |
| Features | MEDIUM-HIGH | Table stakes and differentiators verified against 5+ competitor protocols (GMX V2, Synthetix V3, dYdX V4, Hyperliquid, Perennial). Feature dependencies logical. Anti-features correctly identified (funding rate, cross-margin) based on Inlet's design. Some Injective-specific features (auto-execution, native oracle) less battle-tested but documented. |
| Architecture | MEDIUM | Standard patterns (factory SubMsg+Reply, IndexedMap, oracle queries) HIGH confidence from official CosmWasm docs. Injective-specific integration (OracleQuerier, injective-test-tube) MEDIUM confidence: docs.rs API confirmed, but practical integration details inferred from ecosystem conventions and example contracts. No major unknowns. |
| Pitfalls | MEDIUM-HIGH | CRITICAL pitfalls (stale oracle, rounding, bad debt, precision, gas griefing) verified across OWASP smart contract security, audit reports (Halborn/Oak Security), and real exploits (Perpetual Protocol, IBC reentrancy). Injective-specific pitfalls (crate versioning, test-tube) MEDIUM confidence: based on GitHub issues, changelog analysis, and documentation gaps. Prevention strategies derived from established DeFi best practices. |

**Overall confidence:** MEDIUM-HIGH

The stack, feature set, and core architecture patterns are well-established with HIGH confidence. Injective-specific integration details (oracle module queries, test-tube setup, crate compatibility) are MEDIUM confidence but not blocking -- official APIs are documented, just less battle-tested than generic CosmWasm. No major unknowns that would derail the project. The research provides sufficient foundation for roadmap creation and phase planning.

### Gaps to Address

**During planning/implementation:**

1. **Exact injective-test-tube oracle setup:** The oracle module privilege grant via governance in test-tube is documented but exact API may require experimentation. Recommend creating a reusable test helper function in Phase 1 that encapsulates the setup (grant privilege, relay price). If this proves complex, it's a Phase 1 blocker that needs resolution before proceeding.

2. **Signed arithmetic for PnL:** cosmwasm-std Decimal256 is unsigned. PnL can be negative (losses). Research confirms this is a known issue. Options: (a) use Int256 + scaling, (b) track sign separately, (c) use SignedDecimal from a third-party crate. Recommend deciding in Phase 2 during math library design. Check if injective-math provides a signed decimal type (FPDecimal may support this).

3. **WasmX auto-execution for liquidation:** Injective's unique feature allowing contracts to execute every block. Research confirms it exists and is documented, but gas delegation model needs validation. This is a Phase 6+ feature (post-MVP), but Phase 1 should design the liquidation interface to be compatible (stateless per-position liquidation, not batch).

4. **Insurance fund implementation details:** Research confirms all major protocols have insurance funds. Implementation can be as simple as a balance in factory contract state + admin withdraw. Phase 2 should include the `bad_debt` tracking field in market contract. Phase 5+ can add insurance fund with fee collection. Not a Phase 1 blocker.

5. **Testnet relayer privilege grant:** On injective-888 testnet, the governance process for granting price feeder privileges may be simplified (single validator, fast voting). This needs validation during Phase 1 relayer setup. Document the exact testnet governance flow or check if testnet has permissionless price feeding.

## Sources

### Primary (HIGH confidence)
- [InjectiveLabs/injective-rust](https://github.com/InjectiveLabs/injective-rust) — injective-std source, API, examples
- [InjectiveLabs/cw-injective](https://github.com/InjectiveLabs/cw-injective) — injective-math, example contracts
- [InjectiveLabs/test-tube](https://github.com/InjectiveLabs/test-tube) — injective-test-tube integration testing
- [InjectiveLabs/injective-ts](https://github.com/InjectiveLabs/injective-ts) — @injectivelabs/sdk-ts TypeScript SDK
- [InjectiveLabs/injective-price-oracle](https://github.com/InjectiveLabs/injective-price-oracle) — Price relayer implementation
- [Injective Oracle Module Docs](https://docs.injective.network/developers/modules/injective/oracle) — Native oracle module
- [Injective CosmWasm Docs](https://docs.injective.network/developers-cosmwasm/) — Contract development guides
- [injective-std docs.rs](https://docs.rs/injective-std/1.14.1/injective_std/) — API documentation
- [CosmWasm Book](https://book.cosmwasm.com/) — Actor model, SubMsg, storage patterns
- [cw-storage-plus docs](https://docs.cosmwasm.com/cw-storage-plus/) — IndexedMap, pagination

### Secondary (MEDIUM confidence)
- GMX V2, Synthetix V3, dYdX V4, Hyperliquid, Perennial documentation — Feature analysis, architecture patterns
- [OWASP Smart Contract Top 10 - Oracle Manipulation](https://scs.owasp.org/sctop10/SC02-PriceOracleManipulation/) — Oracle security
- [Cyfrin Liquidation Vulnerabilities](https://www.cyfrin.io/blog/defi-liquidation-vulnerabilities-and-mitigation-strategies) — Liquidation patterns
- [Oak Security CosmWasm Spotlight](https://medium.com/oak-security/cosmwasm-security-spotlight-4-b5ba69b96c5f) — Rounding direction
- [Halborn Mars Protocol Audit](https://www.halborn.com/audits/mars-protocol/) — Address canonicalization
- [CVE-2024-58263 cosmwasm-std](https://security.snyk.io/vuln/SNYK-RUST-COSMWASMSTD-6673770) — Integer overflow fix
- [Perpetual Protocol Liquidation Vulnerability](https://medium.com/chainlight/retrospecting-liquidation-fee-vulnerability-in-perpetual-protocol-c914cadd575a) — Bad debt case study

### Tertiary (LOW confidence)
- Injective Discord, community forums — Anecdotal testnet setup advice (needs validation)
- Margined Protocol (archived) — Reference architecture, but outdated dependencies

---

*Research completed: 2026-02-16*
*Ready for roadmap: YES*
