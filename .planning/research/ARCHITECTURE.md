# Architecture Research

**Domain:** Synthetic Derivatives Engine on Injective (CosmWasm)
**Researched:** 2026-02-16
**Confidence:** MEDIUM (verified against official Injective docs, injective-std API, and CosmWasm patterns; some integration specifics inferred from ecosystem conventions)

## Standard Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         Off-Chain Layer                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐       │
│  │  Price Relayer    │  │  Liquidation Bot │  │  CLI / Scripts   │       │
│  │  (TypeScript)     │  │  (TypeScript)    │  │  (injective-ts)  │       │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘       │
│           │ MsgRelayPrice        │ ExecuteMsg           │                │
│           │ FeedPrice            │ ::Liquidate           │                │
├───────────┴──────────────────────┴──────────────────────┴────────────────┤
│                         Injective Chain                                  │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐   │
│  │                    Oracle Module (native)                         │   │
│  │  MsgRelayPriceFeedPrice → state storage → queryable by contracts │   │
│  └───────────────────────────────────────┬───────────────────────────┘   │
│                                          │ OracleQuerier                 │
│  ┌───────────────────────────────────────┴───────────────────────────┐   │
│  │                    Factory Contract                                │   │
│  │  - Stores market_code_id                                          │   │
│  │  - Instantiates Market contracts via SubMsg + Reply               │   │
│  │  - Maintains registry: Map<String, Addr> (pair → market_addr)     │   │
│  │  - Admin for migrations                                           │   │
│  └────────┬──────────────────────┬──────────────────────┬────────────┘   │
│           │ Instantiate          │ Instantiate           │ Instantiate   │
│  ┌────────▼────────┐   ┌────────▼────────┐   ┌─────────▼───────┐       │
│  │ Market Contract │   │ Market Contract │   │ Market Contract  │       │
│  │ (BTC/USD)       │   │ (ETH/USD)       │   │ (SOL/USD)       │       │
│  │                 │   │                 │   │                  │       │
│  │ - Positions     │   │ - Positions     │   │ - Positions      │       │
│  │ - Collateral    │   │ - Collateral    │   │ - Collateral     │       │
│  │ - PnL calc      │   │ - PnL calc      │   │ - PnL calc       │       │
│  │ - Liquidation   │   │ - Liquidation   │   │ - Liquidation    │       │
│  └─────────────────┘   └─────────────────┘   └──────────────────┘       │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐   │
│  │                    Bank Module (native)                            │   │
│  │  INJ collateral held by each market contract address              │   │
│  └───────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Communicates With |
|-----------|---------------|-------------------|
| **Factory Contract** | Deploy + register market contracts, store `market_code_id`, admin for migrations, global config | Market contracts (instantiate), Chain (admin) |
| **Market Contract** | Position CRUD, collateral custody, PnL calculation, liquidation execution, oracle price queries | Oracle module (query), Bank module (send/receive INJ), Factory (instantiated by) |
| **Oracle Module** (native) | Store + serve price data from authorized relayers | Market contracts (queried by), Relayer (receives price relay msgs) |
| **Price Relayer** (off-chain) | Fetch external prices (Binance, etc.), relay to Injective oracle module | Oracle module (MsgRelayPriceFeedPrice), External price APIs |
| **Liquidation Bot** (off-chain) | Monitor positions, call `Liquidate` when margin < MMR | Market contracts (ExecuteMsg::Liquidate) |
| **Bank Module** (native) | Hold native INJ balances per contract address | Market contracts (BankMsg::Send) |

## Recommended Project Structure

```
inlet/
├── contracts/
│   ├── factory/              # Factory contract
│   │   ├── src/
│   │   │   ├── contract.rs   # instantiate, execute, query, reply
│   │   │   ├── msg.rs        # InstantiateMsg, ExecuteMsg, QueryMsg
│   │   │   ├── state.rs      # CONFIG, MARKETS registry
│   │   │   ├── error.rs      # ContractError
│   │   │   └── lib.rs
│   │   └── Cargo.toml
│   └── market/               # Market contract (one per trading pair)
│       ├── src/
│       │   ├── contract.rs   # instantiate, execute, query
│       │   ├── msg.rs        # InstantiateMsg, ExecuteMsg, QueryMsg
│       │   ├── state.rs      # CONFIG, POSITIONS (IndexedMap), POSITION_SEQ
│       │   ├── oracle.rs     # Oracle query helpers
│       │   ├── position.rs   # Position logic: open, close, PnL
│       │   ├── liquidation.rs# Liquidation logic
│       │   ├── error.rs      # ContractError
│       │   └── lib.rs
│       └── Cargo.toml
├── packages/
│   └── inlet-types/          # Shared types between contracts
│       ├── src/
│       │   ├── factory.rs    # Factory msg types
│       │   ├── market.rs     # Market msg types
│       │   ├── position.rs   # Position struct, Side enum
│       │   └── lib.rs
│       └── Cargo.toml
├── relayer/                  # TypeScript off-chain services
│   ├── src/
│   │   ├── price-relayer.ts  # Price feed → MsgRelayPriceFeedPrice
│   │   ├── liquidation-bot.ts# Position monitoring + liquidation calls
│   │   └── config.ts
│   ├── package.json
│   └── tsconfig.json
├── tests/                    # Integration tests
│   ├── src/
│   │   ├── helpers.rs        # Test setup (deploy factory, create markets, relay prices)
│   │   ├── test_factory.rs   # Factory deployment tests
│   │   ├── test_positions.rs # Open/close position tests
│   │   ├── test_liquidation.rs
│   │   └── lib.rs
│   └── Cargo.toml
├── Cargo.toml                # Workspace root
└── .cargo/
    └── config.toml           # wasm32-unknown-unknown target alias
```

### Structure Rationale

- **contracts/factory + contracts/market:** Separated because they are independently deployed wasm binaries. Each has its own `Cargo.toml` targeting `cdylib`.
- **packages/inlet-types:** Shared message types prevent deserialization mismatches between factory and market. Both contracts depend on this package. This is a standard CosmWasm workspace pattern.
- **relayer/:** Separate TypeScript project. Does not share build with Rust contracts. Uses `@injectivelabs/sdk-ts` for chain interaction.
- **tests/:** Uses `injective-test-tube` for integration tests that run against real chain logic compiled from Go, not mocks. This catches issues that `cw-multi-test` would miss (oracle module behavior, bank module, gas).

## Architectural Patterns

### Pattern 1: Factory + SubMsg + Reply for Child Contract Deployment

**What:** Factory contract uses `WasmMsg::Instantiate` wrapped in a `SubMsg` with `reply_on_success` to deploy market contracts. The `reply` entrypoint captures the new contract address from the instantiation response.

**When to use:** Always for multi-market deployment. The factory must know the address of each market it creates.

**Trade-offs:**
- PRO: Deterministic address capture, atomic deployment
- PRO: Factory becomes the admin of child contracts (can migrate them)
- CON: Slightly more complex than `Instantiate2` for simple cases
- NOTE: `Instantiate2` (predictable addresses) is an alternative but the SubMsg+Reply pattern is more battle-tested and provides a clearer hook for post-instantiation setup

**Confidence:** HIGH -- this is the standard CosmWasm factory pattern used by Tgrade TFi AMM, Margined Protocol, and documented in the CosmWasm official examples.

**Example:**
```rust
// In factory contract.rs

const INSTANTIATE_MARKET_REPLY_ID: u64 = 1;

// Temporary storage for the pair being created (used in reply)
const PENDING_MARKET: Item<String> = Item::new("pending_market");

pub fn execute_create_market(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    base: String,
    quote: String,
) -> Result<Response, ContractError> {
    let config = CONFIG.load(deps.storage)?;

    // Only admin can create markets
    if info.sender != config.admin {
        return Err(ContractError::Unauthorized {});
    }

    let pair_id = format!("{}/{}", base, quote);

    // Check market doesn't already exist
    if MARKETS.has(deps.storage, &pair_id) {
        return Err(ContractError::MarketAlreadyExists { pair: pair_id });
    }

    // Save pending pair for reply handler
    PENDING_MARKET.save(deps.storage, &pair_id)?;

    let instantiate_msg = WasmMsg::Instantiate {
        admin: Some(env.contract.address.to_string()), // factory is admin
        code_id: config.market_code_id,
        msg: to_json_binary(&MarketInstantiateMsg {
            base: base.clone(),
            quote: quote.clone(),
            oracle_type: config.oracle_type,
            max_leverage: config.max_leverage,
            maintenance_margin_ratio: config.maintenance_margin_ratio,
            liquidation_fee_rate: config.liquidation_fee_rate,
        })?,
        funds: vec![],
        label: format!("inlet-market-{}", pair_id),
    };

    let sub_msg = SubMsg::reply_on_success(instantiate_msg, INSTANTIATE_MARKET_REPLY_ID);

    Ok(Response::new()
        .add_submessage(sub_msg)
        .add_attribute("action", "create_market")
        .add_attribute("pair", pair_id))
}

#[entry_point]
pub fn reply(deps: DepsMut, _env: Env, msg: Reply) -> Result<Response, ContractError> {
    match msg.id {
        INSTANTIATE_MARKET_REPLY_ID => {
            let res = cw_utils::parse_instantiate_response_data(
                msg.result.into_result().map_err(StdError::generic_err)?.data
                    .ok_or(ContractError::NoReplyData {})?.as_slice()
            )?;
            let market_addr = deps.api.addr_validate(&res.contract_address)?;
            let pair_id = PENDING_MARKET.load(deps.storage)?;
            PENDING_MARKET.remove(deps.storage);

            MARKETS.save(deps.storage, &pair_id, &market_addr)?;

            Ok(Response::new()
                .add_attribute("action", "market_created")
                .add_attribute("pair", pair_id)
                .add_attribute("market_addr", market_addr))
        }
        _ => Err(ContractError::UnknownReplyId { id: msg.id }),
    }
}
```

### Pattern 2: Oracle Price Query via injective-std Stargate/Any Queries

**What:** Market contracts query the Injective native oracle module for current prices using the `OracleQuerier` from `injective-std`. The oracle module is a native Cosmos SDK module (not a CosmWasm contract), so queries go through the Stargate/Any query mechanism.

**When to use:** Every time a market contract needs the current price -- position opens, closes, PnL calculations, liquidation checks.

**Trade-offs:**
- PRO: No cross-contract call overhead, queries native chain state directly
- PRO: Price data is authoritative (from the oracle module itself)
- CON: Requires `injective-std` dependency and understanding of protobuf query types
- CON: `injective-cosmwasm` (the older JSON-based bindings) is deprecated; must use `injective-std` with protobuf Any encoding

**Confidence:** HIGH -- `OracleQuerier` with `price_feed_price_states()` and `oracle_price()` methods verified in docs.rs/injective-std/latest.

**Example:**
```rust
use injective_std::types::injective::oracle::v1beta1::{
    OracleQuerier, OracleType, QueryOraclePriceRequest,
};

/// Query the current price from Injective's oracle module.
/// Returns the price as Decimal256 or errors if stale/unavailable.
pub fn query_oracle_price(
    deps: Deps,
    config: &MarketConfig,
) -> Result<Decimal256, ContractError> {
    let querier = OracleQuerier::new(&deps.querier);

    // For PriceFeed oracle type:
    let response = querier.oracle_price(
        OracleType::PriceFeed as i32,
        config.base.clone(),       // e.g. "BTC"
        config.quote.clone(),      // e.g. "USD"
        None,                      // scaling_options
    )?;

    let price_pair_state = response.price_pair_state
        .ok_or(ContractError::OraclePriceUnavailable {})?;

    // Parse the price string to Decimal256
    let price = Decimal256::from_str(&price_pair_state.pair_price)?;

    // Staleness guard: reject prices older than threshold
    let current_time = deps.querier.query_block_info()?.time;
    let price_timestamp = price_pair_state.timestamp;
    if current_time.seconds() - price_timestamp > config.max_price_staleness_secs {
        return Err(ContractError::OraclePriceStale {
            last_update: price_timestamp,
            threshold: config.max_price_staleness_secs,
        });
    }

    Ok(price)
}
```

### Pattern 3: Position Storage with IndexedMap + Auto-Incrementing ID

**What:** Positions are stored in an `IndexedMap` keyed by a unique `u64` position ID (auto-incrementing sequence). A `MultiIndex` on `owner` (Addr) allows efficient lookup of all positions for a given user. This is the standard CosmWasm pattern for entities that need lookup by multiple keys.

**When to use:** When users can have multiple positions per market and you need to query by owner or iterate all positions for liquidation checks.

**Trade-offs:**
- PRO: O(1) lookup by position ID, efficient range queries by owner
- PRO: IndexedMap keeps primary and secondary indexes in sync automatically
- CON: Every write updates all indexes (acceptable for position open/close frequency)
- CON: MultiIndex with dynamic-size keys (Addr/String) does not support bounded prefix iteration -- but this is fine since we iterate all positions for a given owner, not a range of owners

**Confidence:** HIGH -- IndexedMap with MultiIndex is the documented CosmWasm pattern (cw-storage-plus official docs, cw721-base reference implementation).

**Example:**
```rust
use cw_storage_plus::{IndexedMap, MultiIndex, IndexList, Index, UniqueIndex};
use cosmwasm_std::{Addr, Uint128, Decimal256, Timestamp};
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

/// Auto-incrementing position ID counter
pub const POSITION_SEQ: Item<u64> = Item::new("position_seq");

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub enum Side {
    Long,
    Short,
}

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct Position {
    pub id: u64,
    pub owner: Addr,
    pub side: Side,
    pub size: Decimal256,        // notional size in base asset
    pub entry_price: Decimal256, // price at open
    pub collateral: Uint128,     // INJ amount in native denom (uinj)
    pub leverage: Decimal256,    // 1x-10x
    pub opened_at: Timestamp,
}

/// Index definitions for positions
pub struct PositionIndexes<'a> {
    pub owner: MultiIndex<'a, Addr, Position, u64>,
}

impl<'a> IndexList<Position> for PositionIndexes<'a> {
    fn get_indexes(&'_ self) -> Box<dyn Iterator<Item = &'_ dyn Index<Position>> + '_> {
        let v: Vec<&dyn Index<Position>> = vec![&self.owner];
        Box::new(v.into_iter())
    }
}

pub fn positions<'a>() -> IndexedMap<'a, u64, Position, PositionIndexes<'a>> {
    let indexes = PositionIndexes {
        owner: MultiIndex::new(
            |_pk, pos: &Position| pos.owner.clone(),
            "positions",
            "positions__owner",
        ),
    };
    IndexedMap::new("positions", indexes)
}
```

### Pattern 4: Bot-Callable Liquidation with Incentive Rewards

**What:** Any external account can call `ExecuteMsg::Liquidate { position_id }`. The contract checks if the position is underwater (collateral + PnL <= maintenance margin requirement). If so, it closes the position, returns remaining collateral minus a liquidation fee to the position owner, and sends the fee to the liquidator as a reward.

**When to use:** For permissionless liquidation. This is standard in DeFi -- incentivize external actors to maintain system health.

**Trade-offs:**
- PRO: No privileged keeper needed, anyone can liquidate
- PRO: Liquidation fee incentivizes monitoring bots
- CON: Race conditions between liquidators (first-to-land wins), but this is acceptable on Cosmos (no MEV concerns like Ethereum)
- NOTE: Start with full liquidation for MVP; partial liquidation adds significant complexity (position splitting, proportional fee calculation)

**Confidence:** MEDIUM -- pattern is well-established in DeFi generally (Aave, Compound, Margined Protocol). Specific Injective implementation details inferred from ecosystem patterns.

**Example:**
```rust
pub fn execute_liquidate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    position_id: u64,
) -> Result<Response, ContractError> {
    let config = CONFIG.load(deps.storage)?;
    let position = positions().load(deps.storage, position_id)?;

    // 1. Get current oracle price
    let current_price = query_oracle_price(deps.as_ref(), &config)?;

    // 2. Calculate PnL
    let pnl = calculate_pnl(&position, current_price)?;

    // 3. Check if position is liquidatable
    // margin_ratio = (collateral + pnl) / notional_value
    let collateral_dec = Decimal256::from_ratio(position.collateral, Uint128::one());
    let notional = position.size * position.entry_price;
    let effective_margin = collateral_dec + pnl; // pnl can be negative (SignedDecimal)

    let maintenance_margin = notional * config.maintenance_margin_ratio;
    if effective_margin > maintenance_margin {
        return Err(ContractError::PositionNotLiquidatable {});
    }

    // 4. Calculate liquidation fee (goes to liquidator)
    let liquidation_fee = notional * config.liquidation_fee_rate;
    let remaining = (collateral_dec + pnl - liquidation_fee)
        .max(Decimal256::zero()); // floor at zero

    // 5. Remove position
    positions().remove(deps.storage, position_id)?;

    // 6. Build bank messages
    let mut msgs = vec![];

    // Pay liquidator their fee
    let fee_amount = liquidation_fee.to_uint_floor();
    if !fee_amount.is_zero() {
        msgs.push(BankMsg::Send {
            to_address: info.sender.to_string(),
            amount: vec![Coin::new(fee_amount.u128(), "inj")],
        });
    }

    // Return remaining collateral to position owner
    let remaining_amount = remaining.to_uint_floor();
    if !remaining_amount.is_zero() {
        msgs.push(BankMsg::Send {
            to_address: position.owner.to_string(),
            amount: vec![Coin::new(remaining_amount.u128(), "inj")],
        });
    }

    Ok(Response::new()
        .add_messages(msgs)
        .add_attribute("action", "liquidate")
        .add_attribute("position_id", position_id.to_string())
        .add_attribute("liquidator", info.sender)
        .add_attribute("fee", fee_amount))
}
```

### Pattern 5: Code ID Management and Contract Migration

**What:** The factory contract stores the `market_code_id` used to instantiate new markets. When upgrading market contract logic, the admin uploads new wasm code (getting a new code_id), updates the factory config, then iterates over all existing markets to send `WasmMsg::Migrate` to each one. The factory is set as the `admin` of each market contract, giving it migration authority.

**When to use:** Every time market contract logic needs updating (bug fix, new feature).

**Trade-offs:**
- PRO: Factory as single migration authority for all markets -- clean governance
- PRO: New markets automatically use latest code_id
- CON: Migration of many markets must be batched (gas limits per tx)
- NOTE: Each market's `migrate` entrypoint must handle state schema changes

**Confidence:** HIGH -- `WasmMsg::Migrate` is a first-class CosmWasm feature. Admin-based migration is documented in cosmwasm-std.

**Example:**
```rust
// Factory: update code_id and migrate all markets
pub fn execute_upgrade_markets(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    new_code_id: u64,
    migrate_msg: Binary,
) -> Result<Response, ContractError> {
    let mut config = CONFIG.load(deps.storage)?;
    if info.sender != config.admin {
        return Err(ContractError::Unauthorized {});
    }

    config.market_code_id = new_code_id;
    CONFIG.save(deps.storage, &config)?;

    // Migrate all existing markets
    let markets: Vec<(String, Addr)> = MARKETS
        .range(deps.storage, None, None, Order::Ascending)
        .collect::<StdResult<Vec<_>>>()?;

    let migrate_msgs: Vec<WasmMsg> = markets
        .iter()
        .map(|(_pair, addr)| WasmMsg::Migrate {
            contract_addr: addr.to_string(),
            new_code_id,
            msg: migrate_msg.clone(),
        })
        .collect();

    Ok(Response::new()
        .add_messages(migrate_msgs)
        .add_attribute("action", "upgrade_markets")
        .add_attribute("new_code_id", new_code_id.to_string())
        .add_attribute("markets_migrated", markets.len().to_string()))
}
```

## Data Flow

### Price Feed Flow

```
External APIs (Binance, CoinGecko, etc.)
    │
    ▼
Price Relayer (TypeScript, @injectivelabs/sdk-ts)
    │ pulls prices on interval (e.g. every 10s)
    │
    ▼
MsgRelayPriceFeedPrice { base, quote, price[] }
    │ broadcast to Injective chain
    │ (relayer must have PriceFeeder privilege via governance)
    │
    ▼
Oracle Module (native Cosmos SDK module)
    │ validates relayer privilege
    │ persists PriceFeedState { base, quote, price, timestamp }
    │
    ▼
Market Contract queries via OracleQuerier::oracle_price()
    │ returns PricePairState with price + timestamp
    │
    ▼
Staleness check: reject if (block_time - price_timestamp) > max_staleness
```

### Position Lifecycle Flow

```
User → ExecuteMsg::OpenPosition { side, size, leverage }
    │ sends INJ as collateral (info.funds)
    │
    ▼
Market Contract:
    1. Query oracle price → entry_price
    2. Validate: leverage <= max_leverage (10x)
    3. Validate: collateral * leverage >= size * price (margin check)
    4. Validate: oracle price is fresh (staleness guard)
    5. Increment POSITION_SEQ → new position_id
    6. Store Position in IndexedMap
    7. Hold INJ in contract balance
    │
    ▼
User → ExecuteMsg::ClosePosition { position_id }
    │
    ▼
Market Contract:
    1. Load position, verify owner == sender
    2. Query oracle price → exit_price
    3. Calculate PnL:
       - Long:  pnl = (exit_price - entry_price) * size
       - Short: pnl = (entry_price - exit_price) * size
    4. payout = collateral + pnl (floor at 0)
    5. Remove position from IndexedMap
    6. BankMsg::Send payout to user
    │
    ▼
Anyone → ExecuteMsg::Liquidate { position_id }
    │
    ▼
Market Contract:
    1. Load position
    2. Query oracle price
    3. Calculate effective margin = collateral + unrealized_pnl
    4. If effective_margin <= maintenance_margin → liquidatable
    5. Pay liquidation_fee to caller
    6. Return remaining to position owner
    7. Remove position
```

### State Management Across Factory + Markets

```
Factory Contract State:
    CONFIG: Item<FactoryConfig>
        - admin: Addr
        - market_code_id: u64
        - oracle_type: i32 (OracleType enum)
        - max_leverage: Decimal256
        - maintenance_margin_ratio: Decimal256
        - liquidation_fee_rate: Decimal256
    MARKETS: Map<String, Addr>         // "BTC/USD" → market_contract_addr
    PENDING_MARKET: Item<String>       // temp storage during SubMsg+Reply

Market Contract State:
    CONFIG: Item<MarketConfig>
        - factory: Addr                // reference back to factory
        - base: String                 // "BTC"
        - quote: String                // "USD"
        - oracle_type: i32
        - max_leverage: Decimal256
        - maintenance_margin_ratio: Decimal256
        - liquidation_fee_rate: Decimal256
    POSITION_SEQ: Item<u64>            // auto-incrementing counter
    POSITIONS: IndexedMap<u64, Position, PositionIndexes>
        - primary key: position_id (u64)
        - secondary index: owner (Addr) via MultiIndex
```

### Key Data Flows

1. **Market Creation:** Admin calls factory `CreateMarket` -> factory sends `WasmMsg::Instantiate` as SubMsg -> `reply` captures address -> saves to `MARKETS` map.
2. **Price Updates:** Relayer fetches external price -> broadcasts `MsgRelayPriceFeedPrice` -> oracle module validates + stores -> market contracts query on demand.
3. **Position Open:** User sends INJ + `OpenPosition` msg -> market queries oracle -> validates margin -> stores position -> holds INJ.
4. **Liquidation:** Bot queries all positions (or specific ones) -> identifies underwater positions -> calls `Liquidate` on market contract -> contract verifies + closes + distributes funds.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 1-10 markets, <100 positions | Current architecture is sufficient. Single relayer, no batching needed. |
| 10-50 markets, <10k positions | Consider pagination for position queries. Liquidation bot should use indexed queries (by owner) rather than full scans. Market migration batching becomes important. |
| 50+ markets, 10k+ positions | Position iteration for liquidation becomes expensive. Consider adding a "liquidation queue" pattern where positions nearing MMR are flagged. Relayer should prioritize high-volume markets. |

### Scaling Priorities

1. **First bottleneck: Liquidation scanning.** Iterating all positions to find liquidatable ones is O(n). Mitigation: liquidation bots maintain their own off-chain index of positions and their margin ratios, only calling on-chain `Liquidate` for known-underwater positions.
2. **Second bottleneck: Oracle staleness under load.** If relayer falls behind, all position operations block on stale price check. Mitigation: run multiple relayer instances, reduce staleness threshold.

## Anti-Patterns

### Anti-Pattern 1: Using injective-cosmwasm Instead of injective-std

**What people do:** Import `injective-cosmwasm` for Injective module bindings because older tutorials reference it.
**Why it's wrong:** `injective-cosmwasm` is deprecated and no longer maintained. It uses JSON-encoded messages instead of protobuf, which is less efficient and incompatible with CosmWasm 2.0+.
**Do this instead:** Use `injective-std` with protobuf-encoded Any messages. Use `OracleQuerier` from `injective_std::types::injective::oracle::v1beta1`.

### Anti-Pattern 2: Storing Prices in Contract State

**What people do:** Have the relayer push prices directly to the market contract via `ExecuteMsg::UpdatePrice`, storing the price in the contract's own state.
**Why it's wrong:** Duplicates the oracle module's job, introduces trust assumptions (who can update?), misses the governance-based privilege system, and creates an extra transaction per market per price update.
**Do this instead:** Relay prices to the native oracle module via `MsgRelayPriceFeedPrice`. Market contracts query the oracle module directly via `OracleQuerier`. The oracle module handles privilege validation, storage, and serves as the single source of truth.

### Anti-Pattern 3: Using Uint128 for Price Calculations

**What people do:** Use `Uint128` for prices and do manual fixed-point math with scaling factors.
**Why it's wrong:** `Uint128` overflows easily with 18-decimal-place token amounts multiplied by prices. Margined Protocol initially used `Uint128` and had to migrate to `Decimal256` (see GitHub issue #15).
**Do this instead:** Use `Decimal256` (18 fractional digits, 256-bit backing) for prices, sizes, and ratios. Use `Uint128` only for native token amounts (collateral in `uinj`). Convert between them explicitly.

### Anti-Pattern 4: Full Position Scan for Liquidation On-Chain

**What people do:** Add a `CheckAllLiquidations` execute message that iterates all positions to find and liquidate underwater ones.
**Why it's wrong:** Gas limit makes this impossible for more than a few positions. On-chain iteration is expensive and unpredictable.
**Do this instead:** Keep liquidation as a per-position call (`Liquidate { position_id }`). Off-chain bots are responsible for identifying which positions to liquidate. The contract only validates and executes the liquidation for a single position.

### Anti-Pattern 5: Storing Contract Address Before Reply

**What people do:** Try to predict or assume the address of a child contract before `reply` returns.
**Why it's wrong:** Contract addresses are not predictable with `WasmMsg::Instantiate` (only with `Instantiate2`). The address is only available in the `reply` handler.
**Do this instead:** Use the `PENDING_MARKET` temp storage pattern: save context before the SubMsg, retrieve it in `reply`, then store the final mapping with the real address.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| **Injective Oracle Module** | `OracleQuerier::oracle_price()` from `injective-std` | Requires `OracleType::PriceFeed` (i32 = 2). Base/quote strings must match what relayer submits. |
| **Injective Bank Module** | Standard `BankMsg::Send`, `info.funds` for receiving | INJ denomination is `"inj"` (or `"uinj"` depending on testnet config -- verify) |
| **Injective Governance** | `GrantPriceFeederPrivilegeProposal` via `MsgSubmitProposal` | Required one-time setup: governance must approve relayer address for each base/quote pair before relayer can submit prices |
| **External Price APIs** | HTTP polling from TypeScript relayer | Use multiple sources (Binance, CoinGecko) and median/mean for robustness |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Factory -> Market | `WasmMsg::Instantiate` (creation), `WasmMsg::Migrate` (upgrade), `WasmQuery::Smart` (optional status queries) | Factory is admin of all markets |
| Market -> Oracle Module | Stargate/Any query via `OracleQuerier` | Read-only, no state change |
| Market -> Bank Module | `BankMsg::Send` for payouts | INJ held in market contract's bank balance |
| Market -> User | Response attributes + `BankMsg::Send` | No callback to user |
| Relayer -> Oracle Module | `MsgRelayPriceFeedPrice` via injective-ts | Off-chain to on-chain, requires governance privilege |
| Liquidation Bot -> Market | `ExecuteMsg::Liquidate` via injective-ts | Off-chain to on-chain, permissionless |

## Injective-Specific Patterns

### Oracle Privilege Setup (Testnet)

Before the relayer can submit prices, a governance proposal must grant price feeder privileges. On testnet, this is typically a simplified process:

```
1. Submit GrantPriceFeederPrivilegeProposal
   - base: "BTC", quote: "USD"
   - relayers: ["inj1...relayer_address"]
2. Vote on proposal (testnet: single validator can pass)
3. Relayer can now send MsgRelayPriceFeedPrice for BTC/USD
4. Repeat for each trading pair (ETH/USD, SOL/USD, etc.)
```

### injective-std vs injective-cosmwasm Decision

Use `injective-std` (not `injective-cosmwasm`). The `injective-std` crate:
- Uses protobuf-encoded messages (aligned with CosmWasm 2.0)
- Is actively maintained
- Provides typed queriers (`OracleQuerier`, `ExchangeQuerier`, etc.)
- Version 1.14.x is current as of early 2026

### Testing with injective-test-tube

The `injective-test-tube` crate provides an `Oracle` module wrapper alongside `Wasm`, `Bank`, `Exchange`, `Gov`, and others. This allows setting up oracle prices in tests by:
1. Creating test accounts with `init_accounts`
2. Using the `Gov` module to grant price feeder privileges to a test account
3. Using the `Oracle` module to relay prices via `MsgRelayPriceFeedPrice`
4. Then executing contract operations that query those prices

This tests against real Injective chain logic compiled from Go, not mocks.

## Decimal Precision Strategy

| Data Type | Rust Type | Rationale |
|-----------|-----------|-----------|
| Prices (entry, exit, oracle) | `Decimal256` | 18 fractional digits, no overflow on multiplication |
| Position size (notional) | `Decimal256` | Fractional sizes allowed, multiplied with prices |
| Leverage | `Decimal256` | 1.0 to 10.0 range |
| Margin ratios (MMR, liquidation fee) | `Decimal256` | Fractional ratios (e.g., 0.05 = 5%) |
| Collateral (INJ amount) | `Uint128` | Native token amounts are integers (in smallest denom) |
| PnL (intermediate calculation) | `SignedDecimal256` or custom | PnL can be negative; cosmwasm-std `Decimal256` is unsigned. Use `Int256` + scaling, or a custom `SignedDecimal` wrapper. |
| Position ID | `u64` | Auto-incrementing integer |

**Critical note on negative PnL:** `Decimal256` is unsigned. For PnL calculations where the result can be negative (unrealized loss), use `cosmwasm_std::Int256` for the signed intermediate, or track sign separately. This is a common source of bugs in CosmWasm DeFi -- plan the signed arithmetic approach before writing position logic.

## Suggested Build Order Based on Dependencies

```
Phase 1: Foundation
├── 1a. Workspace setup (Cargo workspace, .cargo/config.toml, wasm target)
├── 1b. packages/inlet-types (Position, Side, shared msg types)
├── 1c. Market contract skeleton (instantiate, empty execute/query)
└── 1d. Factory contract skeleton (instantiate, empty execute/query/reply)

Phase 2: Oracle Integration
├── 2a. Oracle query helper in market contract (query_oracle_price)
├── 2b. Staleness guard logic
├── 2c. Price relayer (TypeScript, MsgRelayPriceFeedPrice)
└── 2d. injective-test-tube: oracle setup + price query test
    [BLOCKED until oracle helper works]

Phase 3: Position Engine
├── 3a. Position storage (IndexedMap + MultiIndex setup)
├── 3b. OpenPosition logic (margin validation, leverage check, oracle query)
├── 3c. ClosePosition logic (PnL calculation, payout)
├── 3d. Position query endpoints (by ID, by owner)
└── 3e. Integration tests: open → query → close lifecycle
    [DEPENDS ON Phase 2: needs working oracle prices]

Phase 4: Factory + Multi-Market
├── 4a. Factory state (CONFIG, MARKETS, PENDING_MARKET)
├── 4b. CreateMarket with SubMsg + Reply
├── 4c. Market registry queries
├── 4d. Code ID update + migration flow
└── 4e. Integration tests: factory creates market, user opens position
    [DEPENDS ON Phase 3: needs working market contract]

Phase 5: Liquidation
├── 5a. MMR calculation logic
├── 5b. Liquidate execute handler
├── 5c. Liquidation fee distribution
├── 5d. Liquidation bot (TypeScript)
└── 5e. Integration tests: open position → price drops → liquidate
    [DEPENDS ON Phase 3 (positions) + Phase 2 (oracle price manipulation in tests)]

Phase 6: Hardening
├── 6a. Edge cases (zero collateral, max leverage boundary, rounding)
├── 6b. Error handling review
├── 6c. Deviation guards (max price change per update)
├── 6d. Admin controls (pause market, update config)
└── 6e. Full integration test suite
```

**Build order rationale:**
- Phase 1 before everything: workspace structure enables parallel development of contracts.
- Phase 2 before 3: positions cannot be opened without a price. Oracle is the foundational dependency.
- Phase 3 before 4: the market contract must work standalone before the factory can deploy it.
- Phase 4 before 5: liquidation needs multiple markets to be meaningful for testing, though it could technically be built alongside Phase 3.
- Phase 5 last of core features: liquidation requires all prior components (oracle, positions, factory) to be functional.
- Phase 6 is polish: hardening only makes sense after the happy path works.

## Sources

- [Injective Oracle Module Documentation](https://docs.injective.network/developers/modules/injective/oracle) -- HIGH confidence
- [Injective CosmWasm Any Messages](https://docs.injective.network/developers-cosmwasm/cosmwasm-any) -- HIGH confidence
- [injective-std crate on docs.rs](https://docs.rs/injective-std/latest/injective_std/) -- HIGH confidence
- [OracleQuerier API](https://docs.rs/injective-std/latest/injective_std/types/injective/oracle/v1beta1/struct.OracleQuerier.html) -- HIGH confidence
- [OracleType enum](https://docs.rs/injective-std/latest/injective_std/types/injective/oracle/v1beta1/enum.OracleType.html) -- HIGH confidence
- [injective-price-oracle GitHub](https://github.com/InjectiveLabs/injective-price-oracle) -- HIGH confidence
- [injective-test-tube GitHub](https://github.com/InjectiveLabs/test-tube) -- HIGH confidence
- [injective-test-tube Injective Docs](https://docs.injective.network/developers/cosmwasm-developers/injective-test-tube) -- HIGH confidence
- [cw-injective / injective-cosmwasm (deprecated)](https://github.com/InjectiveLabs/cw-injective/tree/dev/packages/injective-cosmwasm) -- MEDIUM confidence (deprecated but useful for understanding module structure)
- [Oracle Governance Proposals](https://docs.injective.network/developers-native/injective/oracle/04_proposals) -- HIGH confidence
- [CosmWasm SubMsg and Reply documentation](https://book.cosmwasm.com/actor-model/contract-as-actor.html) -- HIGH confidence
- [Margined Protocol Perpetuals](https://github.com/margined-protocol/perpetuals) -- MEDIUM confidence (reference architecture, may be outdated)
- [Decimal256 migration discussion](https://github.com/margined-protocol/perpetuals/issues/15) -- MEDIUM confidence
- [cw-storage-plus IndexedMap docs](https://docs.cosmwasm.com/cw-storage-plus/containers/indexed-map) -- HIGH confidence
- [WasmMsg::Migrate in cosmwasm-std](https://docs.rs/cosmwasm-std/latest/cosmwasm_std/enum.WasmMsg.html) -- HIGH confidence
- [Instantiate2 Algorithm](https://cosmwasm.cosmos.network/core/specification/instantiate2-algo) -- MEDIUM confidence (documented but not recommended for this use case)

---
*Architecture research for: Synthetic Derivatives Engine on Injective (CosmWasm)*
*Researched: 2026-02-16*
