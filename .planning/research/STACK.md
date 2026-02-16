# Stack Research

**Domain:** Synthetic derivatives engine on Injective (CosmWasm)
**Researched:** 2026-02-16
**Confidence:** MEDIUM -- injective-std is in active transition (1.14.x stable on crates.io, 1.18.x-beta tracking cosmwasm-std 3.0). Core recommendations are solid, but pin exact versions at project init and recheck before testnet deploy.

---

## Recommended Stack

### On-Chain: Smart Contracts (Rust / CosmWasm)

| Technology | Version | Purpose | Why Recommended | Confidence |
|------------|---------|---------|-----------------|------------|
| **Rust** (stable) | 1.82+ (edition 2021) | Smart contract language | Only production language for CosmWasm. Use latest stable (currently 1.93.1), but pin via `rust-toolchain.toml` for reproducible builds. cw-injective repo pins 1.69.0 which is outdated -- use at least 1.78+ for better async diagnostics and 2021 edition features. | HIGH |
| **cosmwasm-std** | ^2.1.0 | Core CosmWasm SDK | This is the version injective-std 1.14.x depends on. Injective mainnet (v1.16/v1.17) supports CosmWasm 2.0+. Do NOT jump to cosmwasm-std 3.0 yet -- only injective-std beta tracks it. Stick with ^2.1 for stability. | HIGH |
| **injective-std** | 1.14.1 | Injective chain bindings (proto-generated) | **Use this, not injective-cosmwasm.** injective-cosmwasm is deprecated ("no longer maintained, use injective-std"). injective-std uses protobuf-encoded `AnyMsg` instead of JSON, giving better performance, type safety, and forward compatibility with CosmWasm 2.0+. Provides queriers for Exchange, Oracle, Bank, WasmX, TokenFactory, Authz modules. | HIGH |
| **injective-math** | 0.3.0 (stable series) | Fixed-point decimal math (`FPDecimal`) | Critical for DeFi: provides fixed-point arithmetic for price calculations, margin math, and liquidation thresholds. Avoid floating-point entirely in on-chain code. Use `FPDecimal` for all financial calculations. | HIGH |
| **cw-storage-plus** | ^2.0.0 | Storage abstractions (Map, Item, IndexedMap) | Standard CosmWasm storage. Use 2.0.0 (compatible with cosmwasm-std ^2.1). Do NOT use 3.0.0 yet (requires cosmwasm-std 3.0). | HIGH |
| **cw2** | 2.0.0 | Contract versioning/migration support | Standard for tracking contract versions during migrations. Essential for upgradeable factory+market pattern. | HIGH |
| **cosmwasm-schema** | 2.1.x | JSON schema generation for messages | Generates schema files for contract API. Match the cosmwasm-std major version (2.x). | HIGH |
| **serde** | ^1.0 | Serialization/deserialization | Standard Rust serialization. Required by all CosmWasm contracts. | HIGH |
| **thiserror** | ^1.0 | Error handling | Standard error derive macro for CosmWasm contract errors. | HIGH |
| **schemars** | ^0.8.16 | JSON Schema generation | Required by cosmwasm-schema and injective-std for query schema generation. | MEDIUM |

### On-Chain: Testing

| Technology | Version | Purpose | Why Recommended | Confidence |
|------------|---------|---------|-----------------|------------|
| **injective-test-tube** | 1.14.0-rc2 | Integration testing against real chain logic | **Critical for Inlet.** Unlike cw-multi-test, this runs your contract against the actual Injective chain logic (Go modules compiled in). Tests Exchange module interactions (spot/derivative markets), Oracle queries, WasmX auto-execution, and TokenFactory -- all impossible with cw-multi-test. Versioning tracks injective-core: 1.14.x = injective-core v1.14.0. | HIGH |
| **cw-multi-test** | 2.x | Unit testing (simple logic only) | Use for testing pure contract logic (state transitions, access control, math). Insufficient for anything touching Injective-specific modules (Exchange, Oracle, WasmX). Complement with injective-test-tube, not replace. | MEDIUM |

### On-Chain: Build & Deploy

| Tool | Version | Purpose | Notes | Confidence |
|------|---------|---------|-------|------------|
| **cosmwasm/optimizer** | 0.17.0 | Production Wasm compilation | Replaces the old `workspace-optimizer` and `rust-optimizer` images (now unified). Produces deterministic, size-optimized `.wasm` artifacts. Run via Docker. | HIGH |
| **injectived** | v1.17.x | CLI for chain interaction | Deploy, instantiate, execute, query contracts. Use for testnet deployment (`injective-888`). | HIGH |
| **wasm32-unknown-unknown** | (rustup target) | Compilation target | Required Rust target for CosmWasm. Install via `rustup target add wasm32-unknown-unknown`. | HIGH |
| **cargo-generate** | latest | Project scaffolding | Optional but useful for bootstrapping from CosmWasm templates. | LOW |

### Off-Chain: TypeScript Relayer & Frontend

| Package | Version | Purpose | Why Recommended | Confidence |
|---------|---------|---------|-----------------|------------|
| **@injectivelabs/sdk-ts** | ^1.16.12 | Core SDK for Injective chain interaction | The primary SDK for building off-chain components. Provides transaction construction, querying (gRPC/REST), message types for all Injective modules. Actively maintained (published ~Feb 2026). Used by 83+ projects. | HIGH |
| **@injectivelabs/networks** | ^1.17.3 | Network endpoint configuration | Provides pre-configured endpoints for mainnet and testnet (`injective-888`). Eliminates hardcoding RPC/gRPC/LCD URLs. | HIGH |
| **@injectivelabs/ts-types** | ^1.15.41 | TypeScript type definitions | Shared types across the injective-ts ecosystem. Peer dependency of sdk-ts. | HIGH |
| **@injectivelabs/wallet-strategy** | ^1.15.5 | Wallet integration (if building UI) | Replaces deprecated `@injectivelabs/wallet-ts`. Unified interface for Keplr, Leap, MetaMask (EVM), and other wallets. Only needed if Inlet has a frontend. | MEDIUM |
| **Node.js** | 18 LTS or 20 LTS | Runtime | Use LTS versions. The injective-ts monorepo uses `.nvmrc` for version management. | HIGH |
| **TypeScript** | ^5.0 | Language | Standard for the injective-ts ecosystem. | HIGH |

### Oracle: Price Feeds

| Technology | Language | Purpose | Why Recommended | Confidence |
|------------|----------|---------|-----------------|------------|
| **injective-price-oracle** | Go | External price feed submission to chain oracle module | Inlet's price feed infrastructure. Runs as a standalone Go service. Supports dynamic feeds via TOML pipeline configs (HTTP fetch -> JSON parse -> math transforms -> submit) using Chainlink-style pipeline patterns. Also supports custom Go `PricePuller` implementations for complex feeds. Updated Dec 2025. | HIGH |
| **injective-oracle-scaffold** | Go | Local testnet with oracle module | Scaffold for running a full Injective node locally with oracle support. Use for local integration testing of oracle price submission flows. | MEDIUM |

---

## Version Compatibility Matrix

This is the critical compatibility chain. All versions must be compatible with each other.

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| injective-std 1.14.1 | cosmwasm-std ^2.1.0 | Stable release. Tracks injective-core v1.14.0 |
| injective-test-tube 1.14.0-rc2 | injective-std 1.14.x, injective-core v1.14.0 | Version scheme: major tracks injective-core major |
| cosmwasm-std ^2.1.0 | cw-storage-plus 2.0.0, cw2 2.0.0 | The 2.x family is the current stable line for Injective |
| injective-math 0.3.0 | cosmwasm-std ^2.1.0 (0.3.0 stable) | Match with injective-std. Note: 0.3.5-1 exists but tracks cosmwasm-std 3.0 beta |
| @injectivelabs/sdk-ts 1.16.x | @injectivelabs/networks 1.17.x | Keep all @injectivelabs packages on compatible minor versions |
| Injective testnet (injective-888) | injectived v1.17.x | Testnet tracks latest chain version |
| cosmwasm/optimizer 0.17.0 | Rust stable, cosmwasm-std 2.x | Produces deterministic wasm artifacts |

**The golden rule:** Pin injective-std and injective-test-tube to the same major.minor (both 1.14.x). They share the injective-core version they target.

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| **injective-std** (protobuf AnyMsg) | injective-cosmwasm (JSON Stargate) | Never for new projects. injective-cosmwasm is deprecated. Only use if maintaining legacy contracts. |
| **injective-test-tube** | cw-multi-test | Use cw-multi-test for fast unit tests of pure logic. Use test-tube for any test touching Exchange, Oracle, WasmX, or TokenFactory modules. |
| **cosmwasm-std 2.1** | cosmwasm-std 3.0 | Only when injective-std stable releases track 3.0 (currently only in beta 1.18.x). Wait for stable. |
| **@injectivelabs/sdk-ts** | cosmjs (@cosmjs/stargate) | cosmjs works for generic Cosmos chains but lacks Injective-specific message types, Exchange module support, and derivative market abstractions. Use sdk-ts. |
| **injective-price-oracle** (Go) | Custom price feed in TypeScript | injective-price-oracle is battle-tested and supports the full oracle submission flow. Rolling your own in TS means reimplementing gRPC oracle tx submission. Not worth it. |
| **cosmwasm/optimizer 0.17.0** | Manual `cargo build --release` | Never for production. optimizer produces deterministic, stripped, wasm-opt'd binaries. Manual builds are non-deterministic and larger. |
| **cw-storage-plus 2.0** | Raw cosmwasm_std::Storage | cw-storage-plus provides Map, IndexedMap, Item abstractions that dramatically reduce boilerplate. No reason to use raw storage. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **injective-cosmwasm** | Deprecated. Uses JSON-encoded Stargate messages. Will not get new chain features. The maintainers explicitly say "use injective-std". | injective-std 1.14.1 |
| **@injectivelabs/wallet-ts** | Deprecated in favor of wallet-strategy. | @injectivelabs/wallet-strategy |
| **@injectivelabs/sdk-ui-ts** | UI-specific abstractions that couple you to Injective's frontend patterns. For a derivatives engine, you need lower-level control. | @injectivelabs/sdk-ts directly |
| **cosmwasm-std 3.0** | Not yet supported by stable injective-std releases. Using it forces you onto beta bindings (1.18.x-beta) that track unofficial Injective releases. | cosmwasm-std ^2.1.0 |
| **cw-storage-plus 3.0** | Requires cosmwasm-std 3.0. Same issue as above. | cw-storage-plus 2.0.0 |
| **cosmwasm/workspace-optimizer** | Deprecated Docker image. Replaced by unified `cosmwasm/optimizer`. | cosmwasm/optimizer:0.17.0 |
| **cosmwasm/rust-optimizer** | Same -- deprecated, replaced by unified optimizer. | cosmwasm/optimizer:0.17.0 |
| **Floating-point math (f64)** | Non-deterministic across platforms. Will cause consensus failures. Forbidden in CosmWasm. | injective-math FPDecimal, or cosmwasm_std::Decimal256 |
| **CosmWasm template branch 1.0** | Injective docs reference `--branch 1.0` for cw-template, which is outdated. | Use injective-std directly or the cw-injective example contracts as templates. |

---

## Stack Patterns by Variant

**For the factory contract (creates markets):**
- Use `cw2` for contract versioning (migrations when adding new market types)
- Use `cw-storage-plus::Map<MarketId, MarketInfo>` for tracking deployed market contracts
- Use `injective-std` for `MsgInstantiateContract` to deploy market contracts
- No direct Exchange module interaction needed -- factory is administrative

**For each market contract (positions, collateral, liquidations):**
- Use `injective-std` Exchange querier for derivative market data
- Use `injective-std` Oracle querier for price feeds
- Use `injective-math::FPDecimal` for all margin, PnL, and liquidation math
- Use `cw-storage-plus::IndexedMap` for positions (indexed by trader address AND liquidation price)
- Use `injective-std` Bank module for INJ collateral management (native coin)
- Consider WasmX registration for auto-execution (liquidation checks every block)

**For the off-chain relayer (TypeScript):**
- Use `@injectivelabs/sdk-ts` for transaction construction and chain queries
- Use `@injectivelabs/networks` for testnet/mainnet endpoint config
- Use gRPC streaming (via sdk-ts) for real-time position monitoring
- The relayer does NOT need wallet-strategy (server-side, uses private key directly)

**For the oracle price feeder (Go):**
- Use `injective-price-oracle` with TOML dynamic feed configs
- Configure multiple price sources (Binance, Coinbase, etc.) with median aggregation
- Use the `probe` command to validate feed configs before deploying

---

## Installation

### Smart Contract Dependencies (Cargo.toml)

```toml
[package]
name = "inlet-market"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
cosmwasm-std = { version = "2.1", features = ["stargate", "cosmwasm_2_0"] }
cosmwasm-schema = "2.1"
cw-storage-plus = "2.0"
cw2 = "2.0"
injective-std = "1.14.1"
injective-math = "0.3.0"
schemars = "0.8"
serde = { version = "1.0", default-features = false, features = ["derive"] }
thiserror = "1.0"

[dev-dependencies]
injective-test-tube = "1.14.0-rc2"
cosmwasm-schema = "2.1"

[profile.release]
opt-level = "z"
debug = false
rpath = false
lto = true
debug-assertions = false
codegen-units = 1
panic = "abort"
incremental = false
overflow-checks = true
```

### Rust Toolchain (rust-toolchain.toml)

```toml
[toolchain]
channel = "1.82.0"
targets = ["wasm32-unknown-unknown"]
```

### TypeScript Relayer (package.json)

```bash
# Core
npm install @injectivelabs/sdk-ts@^1.16.12 @injectivelabs/networks@^1.17.3 @injectivelabs/ts-types@^1.15.41

# Dev dependencies
npm install -D typescript@^5.0 tsx vitest
```

### Build Commands

```bash
# Compile optimized wasm (from workspace root)
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/optimizer:0.17.0

# Run unit tests
cargo test

# Run integration tests (requires injective-test-tube)
cargo test --test integration

# Deploy to testnet
injectived tx wasm store artifacts/inlet_market.wasm \
  --from=your-key \
  --chain-id=injective-888 \
  --gas=auto --gas-adjustment=1.5 \
  --gas-prices=500000000inj \
  --node=https://testnet.sentry.tm.injective.network:443
```

---

## InjectiveLabs Repository Map

How the key repos relate to each other and to Inlet:

```
InjectiveLabs/injective-rust (injective-std source)
  |
  +-- packages/injective-std       --> Rust bindings (proto-generated)
  |     Used by: your contracts (Cargo.toml dependency)
  |
  +-- packages/injective-std-derive --> Proc macros for injective-std
        Used by: injective-std internally

InjectiveLabs/cw-injective
  |
  +-- packages/injective-cosmwasm  --> DEPRECATED, do not use for new code
  +-- packages/injective-math      --> FPDecimal math library
  |     Used by: your contracts for all financial calculations
  +-- packages/injective-protobuf  --> Proto file generation
  +-- contracts/                   --> Example contracts (atomic orders, dummy)
        Used by: reference/learning

InjectiveLabs/test-tube
  |
  +-- packages/injective-test-tube --> Integration test framework
  |     Used by: your integration tests (dev-dependency)
  +-- packages/test-tube-inj       --> Core test-tube flavor for Injective
        Used by: injective-test-tube internally

InjectiveLabs/injective-ts
  |
  +-- packages/sdk-ts              --> Core TypeScript SDK
  +-- packages/networks            --> Endpoint configuration
  +-- packages/ts-types            --> Shared TypeScript types
  +-- packages/wallet-strategy     --> Wallet integration (if UI needed)
        Used by: your TypeScript relayer

InjectiveLabs/injective-price-oracle (Go)
  |
  Standalone service: submits price feeds to chain oracle module
  Used by: your oracle price feeder deployment

InjectiveLabs/injective-oracle-scaffold (Go)
  |
  Local testnet scaffold with oracle module support
  Used by: local development/testing of oracle flows

InjectiveLabs/injective-chain-releases
  |
  Published binaries for injectived (chain client)
  Used by: testnet/mainnet deployment
```

---

## Key Technical Decisions for Inlet

### 1. Use injective-std AnyMsg pattern, not JSON Stargate

**Why:** injective-cosmwasm is deprecated. The `AnyMsg` pattern (protobuf-encoded messages wrapped in `CosmosMsg::Any`) is the forward-compatible approach. This requires CosmWasm 2.0+ which Injective supports.

**Code pattern:**
```rust
use injective_std::types::injective::exchange::v1beta1::MsgCreateSpotMarketOrder;
use cosmwasm_std::{CosmosMsg, AnyMsg};
use prost::Message;

let msg = MsgCreateSpotMarketOrder { /* fields */ };
let any_msg = AnyMsg {
    type_url: MsgCreateSpotMarketOrder::TYPE_URL.to_string(),
    value: msg.encode_to_vec().into(),
};
let cosmos_msg: CosmosMsg = CosmosMsg::Any(any_msg);
```

### 2. Pin all Injective crate versions to the same chain release

**Why:** injective-std, injective-test-tube, and injective-math all track injective-core releases. Mixing versions (e.g., injective-std 1.14 with injective-test-tube 1.13) causes type mismatches and subtle bugs. Pin all to 1.14.x.

### 3. Use WasmX for auto-executing liquidation checks

**Why:** Injective's WasmX module allows contracts to execute automatically every block without external triggers. Register the market contract with WasmX to check liquidation conditions every block instead of relying on external liquidation bots. This is a unique Injective feature not available on other Cosmos chains.

### 4. Use injective-price-oracle, not a custom TypeScript oracle

**Why:** The Go-based oracle service handles the full oracle submission flow (price fetching, aggregation, transaction signing, gas management). It supports TOML-based dynamic pipeline configs for flexible price source configuration. Building this in TypeScript means reimplementing the gRPC oracle transaction submission protocol.

---

## Open Questions (Validate During Implementation)

1. **injective-std stable vs beta:** The 1.14.1 stable release depends on cosmwasm-std ^2.1.0. The 1.18.x-beta series tracks cosmwasm-std 3.0. For Inlet targeting testnet in the near term, 1.14.1 is the safe choice. Re-evaluate when 1.16+ goes stable on crates.io.

2. **injective-math version alignment:** The stable 0.3.0 on crates.io targets cosmwasm-std ^2.1. The 0.3.5-1 version targets cosmwasm-std 3.0. Ensure you use 0.3.0 with injective-std 1.14.1. Verify exact compatibility at project init by checking Cargo.lock resolution.

3. **WasmX gas delegation:** WasmX auto-execution requires gas. Contracts can delegate gas payment to another address. Investigate the exact gas cost model for per-block liquidation checks across many positions.

4. **Test-tube Exchange module support:** The changelog shows Exchange module message support (`MsgDeposit`, `MsgWithdraw`, `MsgBatchUpdateOrders`) was added progressively. Verify that all derivative market operations needed for Inlet (create market, manage positions, liquidations) are available in 1.14.0-rc2.

5. **Testnet permissionlessness:** Injective testnet (`injective-888`) is permissionless for contract deployment. Mainnet requires governance proposals. Plan mainnet deployment strategy early but build for testnet first.

---

## Sources

- [InjectiveLabs/cw-injective](https://github.com/InjectiveLabs/cw-injective) -- packages for CosmWasm on Injective (HIGH confidence)
- [InjectiveLabs/injective-rust](https://github.com/InjectiveLabs/injective-rust) -- new home for injective-std bindings (HIGH confidence)
- [InjectiveLabs/test-tube](https://github.com/InjectiveLabs/test-tube) -- integration testing framework (HIGH confidence)
- [InjectiveLabs/injective-ts](https://github.com/InjectiveLabs/injective-ts) -- TypeScript SDK monorepo (HIGH confidence)
- [InjectiveLabs/injective-price-oracle](https://github.com/InjectiveLabs/injective-price-oracle) -- Go oracle price feeder (HIGH confidence)
- [InjectiveLabs/injective-oracle-scaffold](https://github.com/InjectiveLabs/injective-oracle-scaffold) -- local oracle testnet scaffold (MEDIUM confidence)
- [injective-std on docs.rs](https://docs.rs/crate/injective-std/latest) -- version 1.14.1, cosmwasm-std ^2.1.0 (HIGH confidence)
- [injective-cosmwasm on lib.rs](https://lib.rs/crates/injective-cosmwasm) -- deprecated notice confirmed (HIGH confidence)
- [injective-test-tube CHANGELOG](https://github.com/InjectiveLabs/test-tube/blob/dev/packages/injective-test-tube/CHANGELOG.md) -- version history and chain targeting (HIGH confidence)
- [Injective docs: Using Queries and Messages](https://docs.injective.network/developers-cosmwasm/cosmwasm-any) -- AnyMsg migration guide (HIGH confidence)
- [Injective docs: Your First Smart Contract](https://docs.injective.network/developers-cosmwasm/smart-contracts/your-first-smart-contract) -- setup instructions (MEDIUM confidence, somewhat outdated optimizer version)
- [Injective docs: Local Development](https://docs.injective.network/developers/cosmwasm-developers/guides/local-development) -- local dev setup (MEDIUM confidence)
- [Injective docs: Test Tube](https://docs.injective.network/developers/cosmwasm-developers/injective-test-tube) -- test-tube usage guide (HIGH confidence)
- [@injectivelabs/sdk-ts on npm](https://www.npmjs.com/package/@injectivelabs/sdk-ts) -- v1.16.12 (HIGH confidence)
- [@injectivelabs/networks on npm](https://www.npmjs.com/package/@injectivelabs/networks) -- v1.17.3 (HIGH confidence)
- [@injectivelabs/wallet-strategy on npm](https://www.npmjs.com/package/@injectivelabs/wallet-strategy) -- v1.15.5, replaces wallet-ts (HIGH confidence)
- [CosmWasm/optimizer](https://github.com/CosmWasm/optimizer) -- unified optimizer v0.17.0 (HIGH confidence)
- [cosmwasm-std on lib.rs](https://lib.rs/crates/cosmwasm-std) -- version 3.0.2 latest, 2.1.x for Injective (HIGH confidence)
- [Injective chain upgrade v1.16.1 docs](https://docs.injective.network/infra/validator-mainnet/canonical-chain-upgrade-1.16.1) -- mainnet version (MEDIUM confidence)
- [Injective v1.17.2 upgrade (IIP-603)](https://x.com/ttt_lab/status/2001885816734519367) -- Dec 2025 mainnet upgrade (MEDIUM confidence, social media source)

---
*Stack research for: Inlet -- synthetic derivatives engine on Injective*
*Researched: 2026-02-16*
