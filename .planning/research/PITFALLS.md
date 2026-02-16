# Pitfalls Research

**Domain:** Synthetic Derivatives Engine on Injective (CosmWasm)
**Researched:** 2026-02-16
**Confidence:** MEDIUM-HIGH (verified against official docs, audit reports, and known exploits; some Injective-specific items rely on fewer sources)

---

## Critical Pitfalls

### Pitfall 1: Stale Oracle Price Exploitation

**Severity:** CRITICAL

**What goes wrong:**
The TypeScript relayer fails, lags, or gets front-run. The on-chain oracle price becomes stale. An attacker opens or closes positions at an outdated price that diverges from the real market price, extracting risk-free profit. Liquidations also fire (or fail to fire) based on stale data. In the worst case, a relayer outage during a flash crash means positions that should be liquidated remain open, accumulating bad debt the protocol cannot recover.

**Why it happens:**
Inlet relies on an off-chain relayer (`injective-price-oracle`) to push prices on-chain. Any gap in this push (relayer downtime, network congestion, gas spike, relayer key compromise) means the contract operates on outdated state. Unlike Chainlink with decentralized node operators, a single relayer is a single point of failure.

**How to avoid:**
1. Store `last_price_update_timestamp` alongside every price write.
2. In every position-opening, position-closing, and liquidation call, assert `block_time - last_price_update_timestamp < MAX_STALENESS` (e.g., 60-120 seconds). Reject the transaction if stale.
3. Implement a circuit breaker: if price deviates more than X% (e.g., 15%) from the previous update in a single relay, reject the update and require manual intervention or multiple confirmations.
4. Add a `pause_market()` admin function that halts all trading when the relayer is known to be down.
5. Consider running multiple relayer instances with different keys for redundancy.

**Warning signs:**
- `last_price_update_timestamp` is not tracked in contract state
- No staleness check exists in `open_position`, `close_position`, or `liquidate`
- Relayer has no health monitoring or alerting
- No integration test simulates "relayer goes down for 5 minutes, then price moves 20%"

**Phase to address:**
Phase 1 (Core contract design). Staleness checks must be part of the initial oracle integration, not bolted on later. Circuit breaker can be Phase 2.

---

### Pitfall 2: Rounding Direction Favoring Users Over Protocol

**Severity:** CRITICAL

**What goes wrong:**
PnL, collateral requirements, and liquidation thresholds are calculated with integer/fixed-point math that inevitably truncates. If truncation consistently favors the user (e.g., collateral requirement rounds down, PnL rounds up), attackers can extract dust amounts per transaction that compound into significant protocol losses. Worse: if liquidation checks round in the user's favor, a position that should be liquidatable may survive, accumulating bad debt.

**Why it happens:**
CosmWasm's `Uint128` and `Decimal` (18-decimal fixed-point) truncate on division. Developers naturally write `collateral / leverage` without thinking about whether the remainder should round up or down. The default truncation (floor) happens to favor different parties depending on context:
- Floor on "how much collateral is needed" = favors user (needs less)
- Floor on "what is the user's PnL" = favors protocol (user gets less)

Missing this means some calculations accidentally donate value to users.

**How to avoid:**
1. Establish a **rounding policy document** before writing any math:
   - Collateral requirements: round UP (user must deposit more, not less)
   - PnL payouts to users: round DOWN (protocol keeps dust)
   - Liquidation threshold checks: round AGAINST the user (position liquidatable sooner, not later)
   - Liquidation fee payouts to liquidators: round DOWN
2. Use explicit CosmWasm methods: `Uint128::checked_mul_ceil()` and `Uint128::checked_div_floor()` (available since CosmWasm 1.2+) rather than bare `/` operator.
3. For every arithmetic function, add a comment stating which party benefits from the rounding direction.
4. Write property-based tests: "the protocol never pays out more than it received" across random position sizes and prices.

**Warning signs:**
- Bare `/` operator used on `Uint128` without considering remainder
- No explicit `_ceil` or `_floor` suffix on division/multiplication calls
- PnL tests only check exact values (never fractional edge cases)
- No test for "open position with 1 unit of collateral at max leverage"

**Phase to address:**
Phase 1 (Core math library). This must be correct from day one. Retrofitting rounding direction is a full rewrite of every formula.

---

### Pitfall 3: Underwater Positions and Bad Debt Without Recovery Mechanism

**Severity:** CRITICAL

**What goes wrong:**
A position's loss exceeds its collateral (`collateral + pnl < 0`), creating negative equity. The liquidator must pay more to close the position than the remaining collateral covers. No rational liquidator will execute this trade, so the position becomes permanently stuck. The protocol is now insolvent by the amount of bad debt. If enough positions go underwater simultaneously (market crash), the entire protocol's collateral pool is insufficient.

**Why it happens:**
Between oracle price updates, the market can move significantly. With 10x leverage, a 10% adverse move wipes out 100% of collateral. If the oracle updates every 10 seconds and a flash crash moves price 15% in that window, the position is already underwater by 5x collateral before any liquidation can trigger. The PROJECT.md describes `collateral + pnl <= MMR threshold` as the liquidation condition, but has no mechanism for `collateral + pnl < 0`.

**How to avoid:**
1. **Always allow liquidation of underwater positions.** When `collateral + pnl < 0`, the liquidator receives whatever collateral remains (even if it is less than the normal liquidation fee). The contract absorbs the difference as bad debt.
2. **Track bad debt explicitly** in contract state: `total_bad_debt: Uint128`. This is an accounting entry that represents protocol losses.
3. **Insurance fund (Phase 2+):** Accumulate a portion of trading fees into a fund that covers bad debt. When bad debt occurs, deduct from insurance fund first.
4. **Auto-Deleveraging (ADL) as last resort (Phase 3+):** If insurance fund is depleted, reduce profitable positions proportionally to cover bad debt. This is how Binance, Hyperliquid, and other mature perp exchanges handle extreme scenarios.
5. **Never let `liquidate()` revert on underwater positions.** If the math would send negative funds to the liquidator, cap at zero and record the shortfall.

**Warning signs:**
- `liquidate()` function subtracts collateral and panics/reverts when result would be negative
- No `bad_debt` field in contract state
- No test case for "price drops 15% in one update with 10x leveraged position"
- The word "insurance" never appears in the codebase

**Phase to address:**
Phase 1 (Liquidation logic). The basic bad debt handling and "always-liquidatable" guarantee must ship with MVP. Insurance fund and ADL are Phase 2-3 enhancements.

---

### Pitfall 4: Precision Mismatch Between Decimal, Uint128, and INJ Denomination

**Severity:** CRITICAL

**What goes wrong:**
INJ uses 18 decimal places (`1 INJ = 10^18 inj` in native denomination). Injective's exchange module uses `FPDecimal` internally. CosmWasm `Decimal` also has 18 fractional digits. `Uint128` has none. Mixing these types without explicit conversion causes off-by-10^18 errors. A position opened with what the developer thinks is "1 INJ" of collateral actually opens with "1 * 10^-18 INJ" (essentially zero), or conversely, a payout of "1" is actually `10^18` INJ, draining the entire contract.

**Why it happens:**
CosmWasm handles native tokens as `Uint128` amounts in their smallest denomination. When the developer writes `Uint128::new(1)` thinking "1 INJ," they actually mean 1 attoINJ (10^-18 INJ). This is easy to get right in simple cases but becomes treacherous when:
- Converting between Decimal (which stores 1.0 as `10^18` internally) and Uint128
- Interfacing with `injective-cosmwasm` queries that may return `FPDecimal` values
- Oracle prices arrive as Decimal strings with variable precision

**How to avoid:**
1. Define a single `PRECISION_FACTOR: Uint128 = Uint128::new(10u128.pow(18))` constant and use it everywhere.
2. Create typed wrappers: `struct Inj(Uint128)` for raw amounts, `struct Price(Decimal)` for prices. Never allow implicit conversion. Force explicit `.to_raw()` / `.from_raw()` methods.
3. Validate that oracle price values are in expected range on every update (e.g., BTC/USD should be > $1000 and < $10,000,000).
4. Write sanity-check tests: "deposit 1 INJ, open position, close at same price, withdraw ~1 INJ" to catch 10^18 errors immediately.
5. Be explicit about units in every variable name: `collateral_raw` (in attoINJ), `collateral_inj` (in INJ as Decimal).

**Warning signs:**
- Variables named `amount` or `collateral` without indicating units
- Direct arithmetic between `Uint128` and `Decimal` without explicit conversion functions
- No unit test that deposits a specific INJ amount and verifies the exact stored value
- Oracle price stored as `String` and parsed ad-hoc in multiple places

**Phase to address:**
Phase 1 (Type system and math library). This is foundational and must be designed before any business logic.

---

### Pitfall 5: Liquidation Race Condition Between Price Update and Liquidation Execution

**Severity:** CRITICAL

**What goes wrong:**
Liquidator bot detects a liquidatable position at price P1. In the same block (or next block), the relayer updates the price to P2 where the position is no longer liquidatable. The liquidation either: (a) executes against stale P1, unfairly liquidating a healthy position, or (b) fails because the contract reads the already-updated P2. Conversely, a position becomes liquidatable at P2 but the liquidation tx was submitted at P1 and arrives in a block where P3 (yet another price) is current.

**Why it happens:**
In Cosmos SDK chains, transactions within a block execute sequentially in a deterministic order set by the validator. The relayer's price update tx and a liquidator's liquidation tx may land in the same block in either order. The contract always reads current state, so the execution order within the block determines which price is "live" during liquidation.

**How to avoid:**
1. **Always use the latest on-chain price at execution time**, not a price passed as a parameter. The liquidator should not supply the price; the contract reads it from its own state.
2. **Never cache prices across submessages.** If the liquidation involves a SubMsg callback, re-read the price in the reply handler.
3. **Accept that liquidation is "at market"**: the liquidation checks health against whatever price is in state at execution time. If the position is healthy at execution time, the liquidation tx simply fails (no penalty for the liquidator, just wasted gas).
4. **Add a `last_liquidation_check_price` return** so liquidator bots know which price was used, enabling them to adjust their strategies.

**Warning signs:**
- Liquidation function accepts a `price` parameter from the caller
- Price is loaded once at the top of a multi-step function and not re-read after state changes
- No test simulating "price update and liquidation in same block"

**Phase to address:**
Phase 1 (Liquidation design). The "always read current state" principle must be enforced from the start.

---

### Pitfall 6: Gas Griefing via Unbounded Position Iteration

**Severity:** CRITICAL

**What goes wrong:**
PROJECT.md specifies "multiple positions per user per market." If a user opens hundreds of tiny positions (1 attoINJ collateral each), any operation that iterates over all of a user's positions (total collateral calculation, batch liquidation, portfolio PnL query) hits the gas limit and reverts. This can make a user's positions impossible to liquidate, creating permanent bad debt.

**Why it happens:**
CosmWasm has block gas limits. Iterating over an unbounded `Map` or `Vec` of positions scales linearly. A malicious user intentionally creates N positions where N * per_position_gas > block_gas_limit. Even well-intentioned users with many small positions can accidentally trigger this.

**How to avoid:**
1. **Enforce a maximum positions-per-user-per-market limit** (e.g., 10-20). Reject `open_position` if the limit is reached.
2. **Liquidation must work per-position, not per-user.** `liquidate(position_id)` should only load and process a single position, not iterate. Each position is independently liquidatable.
3. **Never iterate all positions in an execute message.** Queries (which are read-only) can paginate using `cw-storage-plus` range queries with `.take(limit)`.
4. **Minimum position size:** Require a minimum collateral amount (e.g., 0.01 INJ) to prevent dust positions.

**Warning signs:**
- No `MAX_POSITIONS_PER_USER` constant in the codebase
- `liquidate()` loads a `Vec<Position>` for the user
- Any `for position in all_positions` loop in an execute handler
- No minimum collateral requirement enforced

**Phase to address:**
Phase 1 (Position management). Must be designed into the data model from the start.

---

## High Severity Pitfalls

### Pitfall 7: Reply Handler State Mutation Creating Pseudo-Reentrancy

**Severity:** HIGH

**What goes wrong:**
CosmWasm's actor model prevents classical reentrancy (contract A calls B which calls A again in nested execution). However, when using `SubMsg` with `ReplyOn::Success`, the calling contract's `reply()` entry point is invoked after the sub-message completes. If the developer puts state-changing logic in `reply()` that was supposed to be in `execute()`, an attacker can craft a scenario where the `reply()` handler operates on stale assumptions. Example: execute handler checks collateral, sends a bank transfer via SubMsg, reply handler credits the position. If the bank transfer's success/failure changes state assumptions, the position credit may be incorrect.

**Why it happens:**
Developers coming from Solidity think "CosmWasm has no reentrancy" and stop thinking about execution ordering. They move critical state changes to `reply()` for convenience. The `reply()` handler sees state that was committed before the SubMsg executed, but the SubMsg may have changed external state (bank balances, other contracts).

**How to avoid:**
1. **Checks-Effects-Interactions pattern still applies.** Perform all state validation and state writes in `execute()`. Use `reply()` only for reading the SubMsg result (e.g., new contract address after instantiation).
2. **Minimize SubMsg usage.** For simple bank sends, use `BankMsg::Send` as a terminal message (no reply needed), not as a SubMsg.
3. **If SubMsg is required**, save a "pending operation" to state in `execute()` keyed by the SubMsg `id`. In `reply()`, load the pending operation, validate, and finalize. Never rely on transient state.
4. **Audit every `reply()` handler** for state mutations that depend on pre-SubMsg assumptions.

**Warning signs:**
- `reply()` function contains `.save()` calls to position or collateral state
- SubMsg used for simple fund transfers that do not need a reply
- No `pending_operations` or equivalent interim state pattern
- Developers saying "we don't need to worry about reentrancy in CosmWasm"

**Phase to address:**
Phase 1 (Contract architecture). The SubMsg pattern must be established in the initial contract scaffold.

---

### Pitfall 8: Injective Custom Message/Query Version Drift

**Severity:** HIGH

**What goes wrong:**
The `injective-cosmwasm` crate provides Rust bindings (`InjectiveMsg`, `InjectiveQuery`, `InjectiveQueryWrapper`) for Injective-specific chain modules. When Injective releases a chain upgrade (e.g., v1.13 to v1.14), message and query types may change, fields get added/removed, or behavior changes. The crate version and the on-chain version get out of sync. Contracts compiled with crate v0.2.x against a chain running v1.14 may serialize messages incorrectly, causing silent failures or panics.

**Why it happens:**
Injective upgrades its chain roughly quarterly. The `injective-cosmwasm` crate versions track chain versions but with a lag. During testnet development, the testnet may run a different chain version than what the crate targets. The `injective-test-tube` crate also pins a specific chain version, and when a new chain version is released, `injective-test-tube` must increment its major version for breaking changes.

**How to avoid:**
1. **Pin exact crate versions** in `Cargo.toml`: `injective-cosmwasm = "=0.2.22"`, `injective-test-tube = "=1.13.3"` (or whatever is current).
2. **Check compatibility matrix** before starting: verify which `injective-cosmwasm` crate version corresponds to the current testnet chain version. Document this in a `COMPATIBILITY.md` or in `Cargo.toml` comments.
3. **Subscribe to Injective chain upgrade announcements** (GitHub releases, Discord). Budget time for crate updates after each chain upgrade.
4. **Do not use `*` or `^` version specifiers** for Injective crates.
5. **Test on testnet frequently**, not just at the end. A crate/chain mismatch will manifest as cryptic serialization errors.

**Warning signs:**
- `Cargo.toml` uses `^` or `*` for injective crate versions
- Tests pass locally (test-tube) but fail on testnet
- Serialization errors like "unknown field" or "missing field" on Injective custom messages
- No documentation of which chain version the project targets

**Phase to address:**
Phase 1 (Project setup). Pin versions on day one. Revisit at the start of every phase.

---

### Pitfall 9: Storage Key Namespace Collision in Factory-Spawned Contracts

**Severity:** HIGH

**What goes wrong:**
Each market contract spawned by the factory is an independent contract instance with its own storage. However, within a single market contract, `cw-storage-plus` `Item` and `Map` keys share a flat namespace. If two storage declarations accidentally use the same string key (e.g., `const POSITIONS: Map<...> = Map::new("pos");` and `const CONFIG: Item<...> = Item::new("pos");`), they silently overwrite each other's data, causing corrupted state. This is especially dangerous during contract migration when new storage items are added.

**Why it happens:**
`cw-storage-plus` uses raw byte prefixes derived from the string key. There is no compile-time check for uniqueness. In a large contract with 10+ storage items, or across migration versions where new items are added, a developer may reuse a short prefix accidentally. The collision is silent: no error, just corrupted data that manifests as bizarre deserialization failures or wrong values.

**How to avoid:**
1. **Use a naming convention** that makes collisions obvious: `config_v1`, `positions_v1`, `prices_v1`. The prefix should be descriptive and unique.
2. **Maintain a storage key registry** (a comment block at the top of `state.rs`) listing every key in use.
3. **During migration**, never reuse a key for a different type. If the schema of a Map value changes, use a new key (e.g., `positions_v2`) and migrate data explicitly.
4. **Write a unit test** that collects all storage key strings and asserts uniqueness.
5. **Be cautious with `IndexedMap`**: index prefixes must also be distinct from the primary map prefix and from each other.

**Warning signs:**
- Storage keys are single characters or very short strings ("p", "c", "o")
- No comment block documenting all storage keys
- Migration logic reuses existing keys for new types
- Deserialization errors that appear randomly (some records parse, others don't)

**Phase to address:**
Phase 1 (State design). Establish the naming convention and registry before writing any storage declarations.

---

### Pitfall 10: Missing Minimum Collateral and Leverage Boundary Validation

**Severity:** HIGH

**What goes wrong:**
Without minimum collateral enforcement, users open positions with 1 attoINJ (10^-18 INJ). The position is economically meaningless but consumes storage, gas for liquidation, and creates rounding edge cases. Without maximum leverage validation, a user might open an 11x position if the check is `leverage < MAX_LEVERAGE` instead of `leverage <= MAX_LEVERAGE`. Without minimum size validation, PnL calculations can underflow to zero for every price change, creating "zombie" positions that never get liquidated.

**Why it happens:**
Developers focus on the happy path ("user deposits 100 INJ, opens 5x long") and forget boundary conditions. Validation is seen as boring boilerplate. But every missing boundary is an exploit surface.

**How to avoid:**
1. **Validate every numeric input at the entry point** (in `execute()`, before any state changes):
   - `collateral >= MIN_COLLATERAL` (e.g., 0.01 INJ = 10^16 attoINJ)
   - `1 <= leverage <= MAX_LEVERAGE` (exactly 10, not 10.5)
   - `size > 0`
   - `leverage` is a whole number or allowed increment (e.g., 1x, 2x, ... 10x)
2. **Use Rust's type system**: define `Leverage` as a newtype with a constructor that validates. Do not pass raw `u64` or `Decimal`.
3. **Fuzz test boundaries**: test with collateral = 0, 1, MIN-1, MIN, MAX, MAX+1, u128::MAX.

**Warning signs:**
- `open_position` does not check collateral amount against a minimum
- Leverage is accepted as `Decimal` or `String` without range validation
- No test for zero collateral, zero size, or leverage > 10

**Phase to address:**
Phase 1 (Input validation). Every execute entry point must validate all inputs before proceeding.

---

### Pitfall 11: Access Control Gaps on Oracle Update and Admin Functions

**Severity:** HIGH

**What goes wrong:**
If anyone can call `update_price()`, an attacker pushes a fake price (e.g., BTC = $1), liquidates every position, and drains all collateral. If the `pause_market()` or `update_config()` functions lack proper access control, an attacker can halt the market or change critical parameters like MMR thresholds.

**Why it happens:**
In the rush to ship an MVP, access control is either missing ("we'll add it later") or uses a simple `info.sender == config.admin` check that has no key rotation mechanism. The oracle relayer address may be hardcoded at instantiation with no way to update it if the key is compromised.

**How to avoid:**
1. **Store authorized relayer address(es) in contract config.** Validate `info.sender` against this list on every `update_price()` call.
2. **Implement admin role separation**: `owner` (can change config, pause), `relayer` (can update prices only), `liquidator` (anyone, but only for liquidation). Use `cw-ownable` or a simple role pattern.
3. **Allow relayer key rotation** via an admin-only `update_relayer(new_address)` function.
4. **Consider multi-sig or governance** for critical admin operations (not for testnet MVP, but design the interface to support it).
5. **Test negative cases**: "non-admin calls pause_market() and it reverts."

**Warning signs:**
- `update_price()` does not check `info.sender`
- Single `admin` address with no rotation mechanism
- No test for unauthorized access attempts
- Admin address hardcoded as a string literal instead of loaded from config

**Phase to address:**
Phase 1 (Contract instantiation and auth). Must be correct from the first deploy.

---

## Medium Severity Pitfalls

### Pitfall 12: CosmWasm Integer Overflow in Position Math

**Severity:** MEDIUM

**What goes wrong:**
CosmWasm's `Uint128` arithmetic operators panic on overflow in debug mode and wrap in release mode (pre-fix). Even with the CVE-2024-58263 fix (cosmwasm-std >= 1.4.4), `Uint128` standard operators panic on overflow. A carefully chosen position size near `u128::MAX` could cause a panic in PnL calculation, making the position impossible to close or liquidate.

**Why it happens:**
With 18-decimal precision, "reasonable" amounts get very large. 1 million INJ = `10^24` in raw form. Multiplying two such values without `full_mul()` overflows `u128` (max ~3.4 * 10^38). PnL = `(current_price - entry_price) * size` in raw units can easily overflow if not handled carefully.

**How to avoid:**
1. **Use `Uint256` for intermediate calculations** via `Uint128::full_mul()` which returns `Uint256`. Downcast only the final result.
2. **Use `checked_*` methods everywhere**: `checked_add()`, `checked_sub()`, `checked_mul()`. Never use bare `+`, `-`, `*` on `Uint128` in production code.
3. **Enforce maximum position size** at the input validation layer to keep intermediate values within safe bounds.
4. **Use `multiply_ratio(numerator, denominator)`** for ratio calculations, which internally promotes to `Uint256`.
5. **Pin cosmwasm-std >= 1.5.4** (or latest 2.x) to ensure overflow panics rather than silently wrapping.
6. **Clippy lint**: enable `clippy::arithmetic_side_effects` to catch bare arithmetic.

**Warning signs:**
- Bare `*`, `+`, `-` operators used on `Uint128` values
- No `Uint256` import anywhere in the codebase
- No `checked_*` method calls in math functions
- cosmwasm-std version < 1.4.4

**Phase to address:**
Phase 1 (Math library). Establish the "always use checked ops" convention before writing business logic.

---

### Pitfall 13: Contract Migration Data Loss

**Severity:** MEDIUM

**What goes wrong:**
CosmWasm supports contract migration (uploading new code and pointing an existing contract instance to it). If the new code changes the storage schema (e.g., adds a field to `Position`, changes a Map key type), existing data becomes unreadable. Deserialization fails, and all positions are effectively lost. The migration entry point must explicitly convert old data to the new format.

**Why it happens:**
Developers treat migration like a simple code deploy. They change a struct, redeploy, and all queries fail with "Error parsing into type Position: missing field `new_field`." There is no automatic schema migration in CosmWasm.

**How to avoid:**
1. **Version your state structs**: `PositionV1`, `PositionV2`. Keep both in the codebase during migration.
2. **Write explicit migration logic** in the `migrate()` entry point: load each `PositionV1`, convert to `PositionV2`, save under new key if needed.
3. **For large datasets**, migrate lazily: keep old format readable, convert on first access. Store a `schema_version` in config.
4. **Test migration**: write a test that instantiates with V1 code, populates state, migrates to V2 code, and verifies all data is accessible.
5. **On Injective mainnet**, migration requires governance approval. On testnet, the contract admin can migrate. Ensure the admin is set during instantiation.

**Warning signs:**
- No `migrate()` entry point in the contract
- State structs have no version suffix
- No migration test exists
- Contract instantiated with `admin: None` (making it immutable)

**Phase to address:**
Phase 1 (Contract scaffold). Include `migrate()` entry point from the start, even if it is initially a no-op.

---

### Pitfall 14: Injective Testnet vs. Mainnet Behavioral Differences

**Severity:** MEDIUM

**What goes wrong:**
Code that works on testnet (`injective-888`) fails on mainnet (`injective-1`) due to:
- Different gas costs and limits
- Governance-gated contract deployment on mainnet (permissionless on testnet)
- Different oracle providers available
- Different token denominations and decimal configurations
- Module parameter differences (exchange fees, oracle parameters)

**Why it happens:**
Testnet is designed for developer convenience, not production fidelity. Parameters that are relaxed on testnet (unlimited gas, any account can deploy contracts, simplified oracle) mask issues that surface on mainnet.

**How to avoid:**
1. **Document all testnet assumptions** in a single file. Mark each as "testnet-only" or "expected to differ on mainnet."
2. **Test with mainnet-realistic parameters**: use actual mainnet gas costs in test-tube tests, enforce realistic price ranges.
3. **Design for governance-gated deployment**: ensure contract code is auditable, admin patterns support governance, migration is supported.
4. **Do not hardcode testnet chain-id**, RPC endpoints, or token denominations. Use config/environment variables.
5. **Accept this is testnet prototype**: explicitly mark features that would need changes for mainnet (the project already scopes this correctly).

**Warning signs:**
- Chain ID `injective-888` hardcoded in contract or relayer code
- RPC endpoints hardcoded rather than configured
- Assuming any address can deploy contracts
- No documentation of testnet-specific assumptions

**Phase to address:**
Phase 1 (Project configuration). Externalize configuration from the start, even if only testnet values are used initially.

---

### Pitfall 15: injective-test-tube Setup Complexity and Coverage Gaps

**Severity:** MEDIUM

**What goes wrong:**
`injective-test-tube` runs the actual Injective chain binary in-process, providing realistic testing. But:
1. Build times are extremely long (compiling the Go chain binary).
2. The test environment does not simulate network latency, block propagation, or concurrent transaction ordering.
3. Oracle module setup requires manual privilege grants via governance-like setup, which is cumbersome and error-prone.
4. Limited support for some chain modules (e.g., staking is partially supported).
5. The pinned chain version may not match the current testnet version.

**Why it happens:**
test-tube is a powerful but heavyweight tool. Developers either: (a) avoid it due to build complexity and use only `cw-multi-test` (which does not test Injective-specific features at all), or (b) rely on it exclusively without understanding its limitations.

**How to avoid:**
1. **Use a two-tier testing strategy**:
   - `cw-multi-test` for fast unit-style tests of pure contract logic (math, validation, state transitions)
   - `injective-test-tube` for integration tests that exercise Injective-specific features (oracle module, custom messages)
2. **Cache the WASM build** in CI to avoid recompiling the chain binary on every test run.
3. **Document the oracle setup steps** as a reusable test helper function (grant oracle privileges, register relayer, push initial price).
4. **Supplement with testnet deployment tests** for scenarios test-tube cannot simulate (concurrent txs, real block times).
5. **Pin `injective-test-tube` version** to match the testnet chain version you target.

**Warning signs:**
- Only `cw-multi-test` tests exist, with mocked Injective modules
- Test suite takes > 10 minutes to build from scratch
- Oracle tests skip privilege setup and hardcode prices directly in state
- No testnet deployment validation

**Phase to address:**
Phase 1 (Test infrastructure). Set up both testing tiers before writing contract logic.

---

### Pitfall 16: Price Deviation / Flash Crash Not Handled by Circuit Breaker

**Severity:** MEDIUM

**What goes wrong:**
The relayer pushes a price that is technically "fresh" (within staleness window) but wildly incorrect, either due to: (a) a compromised price source, (b) an exchange API glitch returning $0 or $999,999,999, or (c) an actual flash crash on one exchange that the oracle source over-weights. The contract treats this as a valid price, triggering mass liquidations or allowing positions to be opened at absurd prices.

**How to avoid:**
1. **Max price deviation check per update**: reject any price update where `|new_price - last_price| / last_price > MAX_DEVIATION_PCT` (e.g., 20% for BTC).
2. **Absolute price bounds**: per-market configurable `min_price` and `max_price`. BTC cannot be $0 or $100M.
3. **Require N consecutive updates** to move beyond deviation threshold (smoothing).
4. **Admin override** to manually set price and unpause market after a circuit breaker trip.

**Warning signs:**
- `update_price()` has no comparison with the previous price
- No `max_price_deviation` field in market config
- No test for "oracle reports price = 0"

**Phase to address:**
Phase 2 (Hardening). Basic staleness is Phase 1; deviation checking is a Phase 2 enhancement.

---

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Skip insurance fund | Simpler MVP, less state to manage | Protocol insolvent on first market crash; no bad debt recovery | Testnet only, with explicit "TODO: add insurance fund" and bad debt tracking in place |
| Hardcode relayer address | Faster setup, no admin role logic | Cannot rotate compromised key without contract migration | Testnet only; must add admin key rotation before any real-value deployment |
| Single oracle source | Simpler relayer, one price to validate | Single point of failure for price accuracy | Testnet only; plan for multi-source aggregation architecture |
| Skip funding rate | Simpler PnL model (already scoped out) | Long/short imbalance not economically penalized; perpetual positions have no cost-of-carry | Acceptable for MVP; design position struct to accommodate funding accumulator later |
| `cw-multi-test` only (no test-tube) | Faster test cycles, simpler setup | Injective-specific behavior untested; false confidence | Never acceptable; at minimum, oracle and liquidation flows must use test-tube |
| No pagination on position queries | Simpler query handlers | Queries fail at scale; cannot build reliable frontend or bot | Acceptable for testnet with position limits, but paginate before any production use |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| `injective-price-oracle` relayer | Assuming price updates are atomic and instant; not handling relayer restarts | Implement staleness window, health monitoring, auto-restart with exponential backoff |
| `injective-cosmwasm` crate | Using latest crate version without checking chain compatibility | Pin exact version matching target chain version; check compatibility on every chain upgrade |
| `BankMsg::Send` for collateral payouts | Not checking if the contract has sufficient balance before sending | Query contract balance first or handle the SubMsg error in reply; track internal accounting separately from actual bank balance |
| `cw-storage-plus` `IndexedMap` | Creating many indexes for "convenience" queries | Each index adds write cost; only index what is needed for execute-path lookups. Use off-chain indexing for analytics queries |
| TypeScript relayer (injective-ts) | Hardcoding gas amounts and prices | Use `simulateTx` for gas estimation; implement dynamic gas pricing; handle insufficient balance errors gracefully |
| Factory contract `WasmMsg::Instantiate` | Not storing the created contract address from the reply | Always use SubMsg with reply to capture the new contract address; store it in factory state |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Loading all positions in a single query | Query timeout, out-of-gas | Paginate with `start_after` + `limit` pattern | > 50-100 positions per market |
| Storing full price history on-chain | Storage costs grow linearly; contract state size balloons | Store only latest price + TWAP accumulator; use off-chain indexer for history | > 1000 price updates per market |
| IndexedMap with 3+ indexes on positions | Write operations become expensive | Limit to 1-2 essential indexes; use composite keys for common queries | > 100 writes per block across all markets |
| Factory iterating all markets for admin queries | Gas limit hit on query | Paginate market list; cache aggregated stats separately | > 20-30 markets |
| Decimal arithmetic in hot path | Each Decimal op is ~10x more gas than Uint128 | Pre-compute constants; minimize Decimal operations in PnL calculation; convert to Uint128 early | High-frequency liquidation scenarios |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Oracle price as execute message parameter instead of reading from state | Attacker supplies arbitrary price, drains protocol | Always read price from contract's own state; relayer writes, contract reads |
| No price update sender validation | Anyone pushes fake prices | Check `info.sender` against authorized relayer address stored in config |
| Liquidation bonus exceeds remaining collateral | Transaction reverts, position becomes unliquidatable | Cap liquidation bonus at `min(standard_bonus, remaining_collateral)` |
| Using `addr_validate` on user-supplied addresses without canonicalization | Case-sensitivity issues (uppercase vs lowercase) cause orphaned state | Always `deps.api.addr_validate()` then use the returned `Addr`; never use raw string input as storage key |
| Missing `#[entry_point]` on `migrate` function | Contract becomes immutable; cannot fix bugs post-deploy | Always include `migrate` entry point, even as no-op; set admin during instantiation |
| No position ownership check on close/modify | Any user can close another user's position | Assert `position.owner == info.sender` on every position-modifying operation |

## "Looks Done But Isn't" Checklist

- [ ] **Oracle integration:** Staleness check exists but circuit breaker (deviation check) is missing -- verify both exist
- [ ] **Liquidation:** Happy-path works but underwater position (negative equity) case is unhandled -- verify `collateral + pnl < 0` path exists
- [ ] **Position close:** User can close position but PnL payout does not check contract balance -- verify `BankMsg::Send` amount <= contract balance
- [ ] **Collateral withdrawal:** User can withdraw but no check that withdrawal doesn't make position liquidatable -- verify health check after withdrawal
- [ ] **Factory contract:** Can create markets but cannot pause/decommission a market -- verify admin controls exist
- [ ] **Access control:** Admin functions protected but no key rotation mechanism -- verify `update_admin` exists
- [ ] **Test coverage:** Integration tests exist but all use the same price (no price movement tests) -- verify tests with price changes, especially adverse moves
- [ ] **Error handling:** Contract returns custom errors but they are all `StdError::generic_err` with stringly-typed messages -- verify typed error enum with `thiserror`
- [ ] **Events/Logging:** Positions open and close but no `wasm` events emitted -- verify events for all state transitions (needed for indexing and relayer monitoring)
- [ ] **Contract admin:** Contract is deployed but `admin` field is `None` -- verify admin is set for migration capability

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Stale price exploitation (positions opened at wrong price) | HIGH | Pause market, identify affected positions, manually adjust via migration or admin function; implement staleness check before unpausing |
| Rounding errors favoring users | MEDIUM | Audit all math functions, deploy migration with corrected rounding, positions already closed at wrong amounts cannot be recovered |
| Underwater positions stuck (cannot liquidate) | HIGH | Deploy migration that force-closes underwater positions, absorbs bad debt, adds "always-liquidatable" logic |
| Storage key collision | HIGH | Deploy migration that reads corrupted data, reconstructs correct state under new keys; requires forensic analysis of affected records |
| Injective crate version mismatch | LOW | Update crate version, recompile, redeploy via migration; no data loss, just temporary inability to interact with chain modules |
| Gas griefing via dust positions | MEDIUM | Deploy migration adding position limits and minimum collateral; batch-close existing dust positions via admin function |
| Oracle manipulation (fake price pushed) | CRITICAL | Pause immediately, rotate relayer key, identify and revert affected trades via migration; may require manual state reconstruction |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| P1: Stale oracle price | Phase 1 - Core contracts | Test: advance block time past staleness window, verify open_position reverts |
| P2: Rounding direction | Phase 1 - Math library | Test: property test "protocol never pays out more than received" across 10K random inputs |
| P3: Underwater positions / bad debt | Phase 1 - Liquidation | Test: 10x position, price drops 15%, verify liquidation succeeds and bad_debt is recorded |
| P4: Precision mismatch (INJ 18 decimals) | Phase 1 - Type system | Test: deposit exactly 1 INJ (10^18 raw), verify position collateral matches |
| P5: Liquidation race condition | Phase 1 - Liquidation design | Test: update price and liquidate in same test-tube block; verify correct price used |
| P6: Gas griefing (unbounded iteration) | Phase 1 - Data model | Test: attempt to open MAX+1 positions, verify rejection; liquidate single position by ID |
| P7: Reply handler pseudo-reentrancy | Phase 1 - Contract architecture | Code review: no state mutation in reply() that depends on pre-SubMsg assumptions |
| P8: Injective crate version drift | Phase 1 - Project setup | CI: pin versions, test on testnet after every dependency update |
| P9: Storage key collision | Phase 1 - State design | Test: assert all storage key strings are unique at compile time |
| P10: Missing input validation | Phase 1 - Input validation | Test: fuzz all execute entry points with boundary values (0, MIN-1, MAX+1, u128::MAX) |
| P11: Access control gaps | Phase 1 - Auth design | Test: non-admin calls every admin function, verify all revert |
| P12: Integer overflow | Phase 1 - Math library | Test: calculate PnL with u128::MAX position size, verify checked_* returns error not panic |
| P13: Migration data loss | Phase 1 - Contract scaffold | Test: instantiate V1, populate, migrate to V2, verify all data readable |
| P14: Testnet/mainnet differences | Phase 1 - Configuration | Review: no hardcoded chain IDs, endpoints, or denominations |
| P15: Test infrastructure gaps | Phase 1 - Test setup | Review: both cw-multi-test and test-tube suites exist; oracle flow tested with test-tube |
| P16: Price deviation / circuit breaker | Phase 2 - Hardening | Test: push price 50% different from previous, verify rejection |

## Sources

- [OWASP SC02:2025 - Price Oracle Manipulation](https://scs.owasp.org/sctop10/SC02-PriceOracleManipulation/) - OWASP Smart Contract Security Top 10
- [DeFi Liquidation Vulnerabilities and Mitigation Strategies](https://www.cyfrin.io/blog/defi-liquidation-vulnerabilities-and-mitigation-strategies) - Cyfrin comprehensive liquidation vulnerability catalog
- [CosmWasm Architecture - Actor Model](https://medium.com/cosmwasm/cosmwasm-for-ctos-i-the-architecture-59a3e52d9b9c) - Reentrancy prevention via actor model
- [CosmWasm Floating Point Types](https://book.cosmwasm.com/basics/fp-types.html) - No floating-point in CosmWasm; Decimal/Uint128 only
- [CosmWasm Decimal docs](https://docs.rs/cosmwasm-std/latest/cosmwasm_std/struct.Decimal.html) - 18-decimal fixed-point type
- [CosmWasm Uint128 docs](https://docs.rs/cosmwasm-std/latest/cosmwasm_std/struct.Uint128.html) - checked_mul_ceil, checked_div_floor, full_mul
- [CVE-2024-58263 - Integer overflow in cosmwasm-std](https://security.snyk.io/vuln/SNYK-RUST-COSMWASMSTD-6673770) - Wrapping math vulnerability in pre-1.4.4
- [CosmWasm Map Storage - Pagination](https://book.cosmwasm.com/cross-contract/map-storage.html) - Pagination for gas safety
- [cw-storage-plus Basics - Key Collision](https://cosmwasm.cosmos.network/cw-storage-plus/basics) - Namespace collision risks
- [Oak Security CosmWasm Security Spotlight #4](https://medium.com/oak-security/cosmwasm-security-spotlight-4-b5ba69b96c5f) - Rounding direction: always favor protocol
- [Mars Protocol Audit by Halborn](https://www.halborn.com/audits/mars-protocol/mars-protocol-core-contracts-updated-code-cosmwasm-smart-contract-security-assessment) - Address case sensitivity finding
- [Cosmos IBC Reentrancy Infinite Mint](https://www.asymmetric.re/blog/cosmos-ibc-reentrancy-infinite-mint) - Real-world reentrancy via IBC middleware
- [Injective Price Oracle](https://github.com/InjectiveLabs/injective-price-oracle) - Relayer architecture, governance-gated privilege
- [Injective Oracle Module](https://docs.injective.network/developers/modules/injective/oracle) - On-chain oracle state and TWAP
- [injective-cosmwasm crate](https://docs.rs/injective-cosmwasm) - InjectiveMsg, InjectiveQuery bindings
- [injective-test-tube](https://github.com/InjectiveLabs/test-tube) - Integration testing against real chain logic
- [Injective Mainnet Deployment Guide](https://docs.injective.network/developers/cosmwasm-developers/guides/mainnet-deployment) - Governance-gated deployment on mainnet
- [Perpetual Protocol Liquidation Fee Vulnerability](https://medium.com/chainlight/retrospecting-liquidation-fee-vulnerability-in-perpetual-protocol-c914cadd575a) - Bad debt bypass in ClearingHouse
- [Chainlink Circuit Breakers](https://blog.chain.link/circuit-breakers-and-client-diversity-within-the-chainlink-network/) - Price deviation protection patterns
- [Ensuring Safe Oracle Usage in DeFi](https://hackmd.io/@tapir/S17-GbR-3) - Comprehensive oracle safety best practices
- [CosmWasm Metering Security Issues](https://medium.com/cosmwasm/metering-is-hard-cosmwasm-security-issues-explained-a797511cd54e) - Gas metering edge cases

---
*Pitfalls research for: Synthetic Derivatives Engine on Injective (CosmWasm)*
*Researched: 2026-02-16*
