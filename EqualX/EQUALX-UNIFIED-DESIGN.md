# EqualX v1 — Unified Design Document

**Version:** 1.0  
**Status:** Comprehensive Reference (Code-Verified)  
**Last Updated:** December 2025  
**Source of Truth:** Smart contracts in `contracts/` directory

---

## Executive Summary

EqualX is a deterministic, MEV-resistant settlement protocol built on **Maker Auction Markets (MAMs)** — a novel market structure where liquidity emerges from fully collateralized Dutch auction curves published by Makers. The protocol replaces AMMs, solver networks, and oracles with explicit commitments, enabling single-curve settlement with hard user constraints enforced onchain.

> **Note:** This document reflects the **current implementation** as verified against the smart contract source code. Some older documentation files may describe features that have been simplified or removed.

### Core Innovation

Instead of reactive pricing (AMMs) or privileged routing (solvers), EqualX uses **deterministic Dutch auctions** where:
- Makers publish immutable price schedules
- Users choose which auction to fill
- Settlement is atomic and MEV-resistant
- All collateral is fully reserved upfront

### Key Properties

| Property | Guarantee |
|----------|-----------|
| **Determinism** | Price predetermined by curve, not pool state |
| **Solvency** | All auctions 100% collateralized via Strategy Buckets |
| **User Safety** | Constraints enforced before any transfers |
| **MEV Resistance** | No sandwich attacks possible |
| **Maker Sovereignty** | Complete control over inventory and pricing |
| **Accessibility** | Small Makers compete on curve quality, not capital |

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [Maker Auction Markets (MAMs)](#2-maker-auction-markets-mams)
3. [Core Contracts](#3-core-contracts)
4. [Desk Model](#4-desk-model)
5. [Auction Lifecycle](#5-auction-lifecycle)
6. [Settlement Flow](#6-settlement-flow)
7. [Fee Model](#7-fee-model)
8. [Execution Modes](#8-execution-modes)
9. [Specialized Desks](#9-specialized-desks)
10. [Cross-Chain Atomic Swaps](#10-cross-chain-atomic-swaps)
11. [Security Model](#11-security-model)
12. [Off-Chain Components](#12-off-chain-components)
13. [Gas Efficiency](#13-gas-efficiency)
14. [Composability](#14-composability)
15. [Appendices](#appendices)

### Code Verification Summary

This document was verified against the following source files:

| Component | Source File | Verified |
|-----------|-------------|----------|
| Router | `contracts/core/Router.sol` | ✓ |
| DeskVault | `contracts/core/DeskVault.sol` | ✓ |
| AuctionHouse | `contracts/core/AuctionHouse.sol` | ✓ |
| Types | `contracts/lib/Types.sol` | ✓ |
| MathLib | `contracts/lib/MathLib.sol` | ✓ |
| AtomicDesk | `contracts/atomic/AtomicDesk.sol` | ✓ |
| SettlementEscrow | `contracts/atomic/SettlementEscrow.sol` | ✓ |
| OptionsManager | `contracts/options/OptionsManager.sol` | ✓ |

---

## 1. System Architecture

### 1.1 Three Core Contract Systems

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   DeskVault     │◄──►│  AuctionHouse   │◄──►│     Router      │
│                 │    │                 │    │                 │
│ • Collateral    │    │ • Curve Registry│    │ • Settlement    │
│ • Balances      │    │ • Pricing       │    │ • Constraints   │
│ • Reservations  │    │ • Lifecycle     │    │ • Execution     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                │
                    ┌───────────────────────┐
                    │    Users / Makers     │
                    └───────────────────────┘
```

#### DeskVault — Collateral Management
- Per-maker, per-pair inventory model
- Tracks free, reserved, and fee balances
- Guarantees solvency for all auctions
- Supports both ERC20 and native ETH as first-class assets
- Strategy Buckets decouple collateral reservation from curve creation

#### AuctionHouse — Curve Registry & Lifecycle
- Stores minimal onchain state (commitment hash, remaining volume, generation)
- Full curve parameters provided in calldata
- Commitment-based verification prevents tampering
- Supports adaptive curve updates (ADR-004)

#### Router — Settlement Engine
- Single entry point for all swaps
- Enforces user constraints before any transfers
- Direct execution only (delegated/trusted modes removed)
- Handles both ERC20 and ETH settlement
- Atomic settlement with checks-effects-interactions pattern

### 1.2 Integration Flow

```
User Intent → Router → AuctionHouse (load curve) → DeskVault (settle)
                ↓
         Verify constraints
         Compute price
         Check collateral
         Execute transfers
         Update state
         Emit events
```

---

## 2. Maker Auction Markets (MAMs)

### 2.1 What is a MAM?

A market structure where:
1. **Makers** publish deterministic Dutch auction curves
2. **Curves** specify exact price schedules over time
3. **Users** choose which curve to fill
4. **Settlement** is atomic against one curve per transaction

### 2.2 Comparison with Existing Models

| Aspect | AMM | Orderbook | RFQ | MAM |
|--------|-----|-----------|-----|-----|
| Price Formation | Pool reserves | Order matching | Solver | Maker commitment |
| MEV Exposure | High (sandwich) | Medium | High | None |
| Collateral | Pooled | Per-order | Per-quote | Per-curve |
| Maker Control | Limited | High | High | High |
| User Protection | Slippage | Partial fill | Opaque | Absolute |
| Scalability | Good | Poor | Centralized | Excellent |

### 2.3 Key Advantages

1. **Deterministic Pricing**: Price cannot change mid-auction
2. **No Sandwiching**: No pool state to manipulate
3. **Maker Sovereignty**: Complete control over curves and inventory
4. **User Safety**: Strict constraint enforcement
5. **MEV Resistance**: Only transaction ordering matters
6. **Accessibility**: Small Makers compete on curve quality, not capital

### 2.4 Flat Price Auctions as Limit Orders

A Dutch auction with equal start and end prices behaves exactly like a limit order:
- Fills only at that price
- Unfilled portion can be cancelled
- Multiple flat auctions create discrete orderbook depth

---

## 3. Core Contracts

### 3.1 Contract Overview

| Contract | Location | Purpose |
|----------|----------|---------|
| **DeskVault** | `contracts/core/DeskVault.sol` | Collateral management |
| **AuctionHouse** | `contracts/core/AuctionHouse.sol` | Curve registry |
| **Router** | `contracts/core/Router.sol` | Settlement engine |
| **AmmAuctionManager** | `contracts/core/AmmAuctionManager.sol` | Constant-product desks |

### 3.2 Library Contracts

| Library | Purpose |
|---------|---------|
| **MathLib** | Price computation, fixed-point math |
| **CurveHasher** | Commitment hash computation |
| **AmmMath** | Constant-product math |
| **Types** | Shared data structures |
| **Errors** | Revert reasons |

### 3.3 Key Data Structures (From contracts/lib/Types.sol)

```solidity
/// @notice Execution side relative to Desk base asset.
enum Side {
    SellAForB,
    SellBForA
}

/// @notice Fee asset the auction expects.
enum FeeAsset {
    TokenIn,
    TokenOut  // Note: TokenOut is NOT supported in current implementation
}

/// @notice Direct (msg.sender-bound) execution call (minimal surface).
struct DirectIntent {
    uint256 curveId;
    uint128 amountIn;
    uint128 minAmountOut;
    uint64 userDeadline;
}

/// @notice Canonical descriptor containing every parameter of a commitment curve.
struct CurveDescriptor {
    bytes32 deskId;
    uint256 bucketId;
    address tokenA;
    address tokenB;
    bool side;                // false: SellAForB, true: SellBForA
    bool priceIsQuotePerBase;
    uint128 maxVolume;
    uint128 startPrice;
    uint128 endPrice;
    uint64 startTime;
    uint64 duration;
    uint32 generation;
    uint16 feeRateBps;
    FeeAsset feeAsset;        // Must be TokenIn in current implementation
    uint16 supportBps;        // Must be 0 in current implementation
    address supportAddress;   // Must be address(0) in current implementation
    uint96 salt;
}

/// @notice Minimal onchain representation for a committed curve.
struct StoredCurve {
    bytes32 commitment;
    uint128 remainingVolume;
    uint64 endTime;
    uint32 generation;
    bool active;
}

/// @notice Mutable-only parameters permitted during curve updates.
struct CurveUpdateParams {
    uint128 startPrice;
    uint128 endPrice;
    uint64 startTime;
    uint64 duration;
}
```

### 3.4 DeskVault Structures

```solidity
struct Balances {
    uint256 freeA;
    uint256 reservedA;
    uint256 freeB;
    uint256 reservedB;
}

struct MakerFees {
    uint256 feeA;
    uint256 feeB;
}

struct StrategyBucket {
    bytes32 deskId;
    address maker;
    address token;      // Base asset for this bucket
    uint256 reserved;   // Total reserved from free balances
    uint256 remaining;  // Remaining executable base
    bool baseIsA;
    bool active;
}
```

---

## 4. Desk Model

### 4.1 Desk Identification

```solidity
deskId = keccak256(abi.encodePacked(maker, tokenA, tokenB))
```

Each Desk is a sovereign trading lane for a specific asset pair. Either token may be `address(0)` for native ETH.

### 4.2 Balance Model

For each Desk and asset:

| Balance Type | Description |
|--------------|-------------|
| **Free** | Withdrawable, available for new auctions |
| **Reserved** | Locked by active auctions |
| **Fee** | Earned from fills, always withdrawable |

**Core Invariant:**
```
free + reserved + fees = actual balance
```

### 4.3 Collateral Lifecycle

```
Deposit → Reserve → Fill/Cancel → Withdraw
   ↓         ↓          ↓            ↓
 +free    free→res   res→fees    -free
```

1. **Deposit**: Maker funds Desk (increases free balance)
2. **Reserve**: Auction creation moves funds to reserved
3. **Fill**: Settlement decreases reserved, increases fees
4. **Cancel**: Unused portion returns to free balance
5. **Withdraw**: Maker extracts free balance or fees

### 4.4 ETH as First-Class Asset

- Represented by `address(0)` in token slots
- No wrapping/unwrapping required
- Same accounting as ERC20 tokens
- Direct `call{value: ...}` transfers after state updates

---

## 5. Auction Lifecycle

### 5.1 Creation

**Validation Requirements:**
- Bucket must exist and be active
- Maker must own the bucket
- Base asset must match bucket token
- `generation == 1` for new curves
- Nonzero prices, duration, and maxVolume

**Process:**
1. Compute `commitment = CurveHasher.curveHash(desc)`
2. Store `StoredCurve` with commitment and metadata
3. Bind curve to bucket via `curveBucket[curveId]`
4. Emit `CurveCreated` event

### 5.2 Pricing Function

```
if t <= startTime:
    price = startPrice
else if t >= startTime + duration:
    price = endPrice
else:
    price = startPrice - (startPrice - endPrice) * (t - startTime) / duration
```

Linear interpolation ensures deterministic, predictable pricing.

### 5.3 Adaptive Updates (ADR-004)

Makers can update time/price parameters in-place:
- Generation counter increments
- New commitment hash computed
- Immutable fields (bucket, side, maxVolume) unchanged
- Enables HFT-style behavior while preserving determinism

### 5.4 Cancellation

- Only affects remaining volume
- Returns unused collateral to free balance
- Does not affect settled fills
- Auction becomes inactive

---

## 6. Settlement Flow

### 6.1 Router Entry Points (Current Implementation)

The Router has been **simplified** to direct-only execution with minimal intent structure:

#### ERC20 Swaps
```solidity
function executeDirect(DirectIntent calldata intent) external payable
```
- Caller is the user (`msg.sender`)
- No signature verification
- Immediate execution

#### ETH Swaps
```solidity
function executeDirectEth(DirectIntent calldata intent) external payable
```
- For swaps involving ETH as quote or base
- Validates `msg.value` matches required ETH

#### Options/Futures Integration
```solidity
function executeDirectFor(address user, DirectIntent calldata intent) external payable
function executeDirectEthFor(address user, DirectIntent calldata intent) external payable
function executeDirectForFutures(address user, DirectIntent calldata intent) external payable
function executeDirectEthForFutures(address user, DirectIntent calldata intent) external payable
```
- Only callable by configured OptionsManager or FuturesManager
- Executes on behalf of option/futures holders

### 6.2 DirectIntent Structure (Minimal)

```solidity
struct DirectIntent {
    uint256 curveId;       // Target curve identifier
    uint128 amountIn;      // Quote amount to spend
    uint128 minAmountOut;  // Minimum base to receive
    uint64 userDeadline;   // Execution deadline
}
```

**Removed from earlier designs:**
- ~~`user` field~~ (caller is always user)
- ~~`slippageBps`~~ (use `minAmountOut` for protection)
- ~~`referencePriceX18`~~ (removed)
- ~~`deskScope`~~ (removed)
- ~~`feeAssetAllowed`~~ (TokenIn only)

### 6.3 Settlement Sequence

1. **Load Curve Data**
   ```solidity
   CurveFillView memory curveView = auctionHouse.loadCurveForFill(intent.curveId);
   ```
   - Retrieves all curve metadata from AuctionHouse
   - No calldata descriptor verification (simplified)

2. **Validate Constraints**
   - `block.timestamp <= userDeadline`
   - `amountIn > 0 && minAmountOut > 0`
   - Curve is active and within time bounds
   - Desk is not atomic-enabled (atomic desks use separate path)

3. **Compute Settlement**
   ```solidity
   uint256 price = MathLib.computePrice(
       curveView.startPrice,
       curveView.endPrice,
       curveView.startTime,
       curveView.duration,
       block.timestamp
   );
   uint256 baseFill = MathLib.amountOutForFill(amountIn, price);
   ```

4. **Calculate Fees (TokenIn Only)**
   ```solidity
   uint256 makerFee = MathLib.computeFeeBps(amountIn, feeRateBps);
   uint256 totalQuoteNeeded = amountIn + makerFee;
   ```

5. **Verify Fill**
   - `baseFill > 0` (no zero fills)
   - `baseFill <= remainingVolume`
   - `baseFill >= minAmountOut`

6. **Consume Curve & Bucket**
   ```solidity
   auctionHouse.consumeCurveAndBucket(curveId, uint128(baseFill));
   ```

7. **Execute Transfers**
   - Pull quote from user (ERC20 via `safeTransferFrom` or ETH via `msg.value`)
   - Call `vault.settleSwap()` to:
     - Reduce reserved base
     - Credit maker fees
     - Credit quote to maker's free balance
     - Send base to user

### 6.4 Atomicity Guarantee

All-or-nothing execution:
- If any step fails, entire transaction reverts
- No partial fills
- No orphaned state
- Consistent invariants maintained

---

## 7. Fee Model

### 7.1 Fee Recipients

Only **Makers** earn fees from settlement. Protocol collects nothing.

### 7.2 Fee Asset (Current Implementation)

**TokenIn fees ONLY** - The current implementation enforces:
```solidity
if (desc.feeAsset != FeeAsset.TokenIn) revert Errors.InvalidParam();
```

TokenOut fees and support donations have been **removed** from the current codebase.

### 7.3 Fee Calculation

```solidity
uint256 makerFee = MathLib.computeFeeBps(amountIn, feeRateBps);
uint256 totalQuoteNeeded = amountIn + makerFee;
```

- Fee is calculated on the quote (input) amount
- Added to `totalQuoteNeeded` that user must provide
- Credited to maker's fee balance in DeskVault

### 7.4 Removed Features

The following fee features from earlier designs are **not in the current implementation**:

- ~~TokenOut fees~~ (removed)
- ~~Optional support donations~~ (removed - `supportBps` and `supportAddress` must be 0)
- ~~Delegated execution tips~~ (removed - no delegated execution)

---

## 8. Execution Modes

### 8.1 Current Implementation: Direct Only

The Router has been **simplified to direct execution only**. Earlier designs included delegated (EIP-712 signed) execution, but this has been removed.

#### Available Entry Points

| Function | Purpose | Caller |
|----------|---------|--------|
| `executeDirect` | ERC20 swaps | User (msg.sender) |
| `executeDirectEth` | ETH swaps | User (msg.sender) |
| `executeDirectFor` | Options exercise | OptionsManager only |
| `executeDirectEthFor` | Options exercise (ETH) | OptionsManager only |
| `executeDirectForFutures` | Futures settlement | FuturesManager only |
| `executeDirectEthForFutures` | Futures settlement (ETH) | FuturesManager only |

### 8.2 Removed Features

The following execution features from earlier designs are **not implemented**:

- ~~Delegated execution~~ (EIP-712 signed orders)
- ~~Nonce-based replay protection~~
- ~~Executor tips~~
- ~~Trusted executors mapping~~
- ~~Slippage bounds (slippageBps, referencePriceX18)~~
- ~~Desk scoping (deskScope)~~
- ~~Fee asset selection (feeAssetAllowed)~~

### 8.3 User Protection

Users are protected via:
- `minAmountOut` - absolute minimum output required
- `userDeadline` - transaction must execute before this timestamp
- Atomic settlement - all-or-nothing execution

---

## 9. Specialized Desks

### 9.1 LaunchDesk — Token Launches

**Purpose:** Standardized liquidity bootstrapping for new tokens

**Features:**
- One-sided token deposit
- Dutch auction-based price discovery
- Optional withdrawal locks (monotonic)
- Maker fees always withdrawable during lock

**Auction Templates:**
- Classic descending curve (price discovery)
- Ascending commitment curve (early-buyer advantage)
- Tranche series (staged supply release)

### 9.2 Options Desk — Tokenized Derivatives

**Purpose:** Trustless options using ERC-1155 tokens

**Components:**
- **OptionsToken**: ERC-1155 representing exercise rights
- **OptionsManager**: Creates series backed by existing strike curves

**Current Implementation:**

```solidity
struct Series {
    address maker;
    address underlying;
    address payment;
    uint128 strike;      // quote per base, 1e18
    uint64 expiry;
    uint128 totalSize;
    uint128 remaining;
    bool american;
    uint256 curveId;     // References existing flat-price curve
    bool baseIsA;
}
```

**Key Functions:**
- `createSeries(SeriesParams)` - Create option series backed by existing curve
- `exercise(holder, seriesId, amount, minAmountOut, deadline)` - Exercise options
- `reclaim(seriesId)` - Maker reclaims unused collateral after expiry

**Settlement:**
- Exercise calls `router.executeDirectFor()` or `router.executeDirectEthFor()`
- Burns option tokens via `optionToken.managerBurn()`
- Settles against the flat-price strike curve

**Variants:**
- American (exercise anytime before expiry)
- European (exercise only at expiry)

### 9.3 Futures Desk — Forward Contracts

**Purpose:** Tokenized forward contracts

**Components:**
- **FuturesLongToken**: ERC-1155 representing long position
- **FuturesManager**: Manages collateral and settlement

**Properties:**
- Fully collateralized
- No liquidation cascades
- Deterministic settlement
- Oracle-free pricing

### 9.4 AMM Auction Manager

**Purpose:** Isolated constant-product desks (ADR-002)

**Properties:**
- Separate from Router/MAM settlement
- Direct `swapExactIn` interface
- Deterministic x*y=k pricing
- Fully collateralized inventory
- Maker-owned, non-pooled

---

## 10. Cross-Chain Atomic Swaps

### 10.1 Overview

Atomic Desks enable trustless ETH/ERC20 ↔ XMR swaps using cryptographic adaptor signatures.

### 10.2 Components (Current Implementation)

| Contract | Location | Purpose |
|----------|----------|---------|
| **AtomicDesk** | `contracts/atomic/AtomicDesk.sol` | Reservation entry point |
| **SettlementEscrow** | `contracts/atomic/SettlementEscrow.sol` | Holds collateral, manages settlement/refund |
| **Mailbox** | `contracts/atomic/Mailbox.sol` | Encrypted communication channel |
| **EncPubRegistry** | `contracts/atomic/EncPubRegistry.sol` | Decentralized key exchange |

### 10.3 AtomicDesk Contract

```solidity
struct DeskConfig {
    address maker;
    address tokenA;
    address tokenB;
    bool baseIsA;
    bool active;
}

struct ReservationMeta {
    bytes32 deskId;
    address asset;
    uint64 expiry;
    bool initialized;
    bool active;
}
```

**Key Functions:**
- `registerDesk(tokenA, tokenB, baseIsA)` - Register atomic desk
- `reserveAtomicSwap(deskId, taker, asset, amount, settlementDigest, expiry)` - Create reservation
- `setHashlock(reservationId, hashlock)` - Set hashlock commitment

**Constraints:**
- `MIN_EXPIRY_WINDOW = 5 minutes`
- Expiry must be within `refundSafetyWindow`
- Taker cannot be maker
- Desk must be atomic-enabled in DeskVault

### 10.4 SettlementEscrow Contract

```solidity
enum ReservationStatus {
    None,
    Active,
    Settled,
    Refunded
}

struct Reservation {
    uint256 reservationId;
    bytes32 deskId;
    address desk;
    address taker;
    address tokenA;
    address tokenB;
    bool baseIsA;
    address asset;
    uint256 amount;
    bytes32 settlementDigest;
    bytes32 hashlock;
    uint256 auctionId;
    uint64 createdAt;
    ReservationStatus status;
}
```

**Key Functions:**
- `reserve(...)` - Create escrow reservation (called by AtomicDesk)
- `setHashlock(reservationId, hashlock)` - Set hashlock (once only)
- `settle(reservationId, tau)` - Settle by revealing adaptor secret
- `refund(reservationId, noSpendEvidence)` - Committee refund after safety window

**Access Control:**
- `settle`: Only desk maker or committee members
- `refund`: Only committee members
- `setHashlock`: Only reservation operators (Router or AtomicDesk)

### 10.5 Protocol Flow

```
1. Maker enables atomic desk in DeskVault
2. Maker registers desk via AtomicDesk.registerDesk()
3. Maker creates reservation via AtomicDesk.reserveAtomicSwap()
   - Deposits collateral into DeskVault
   - Reserves inventory
   - Creates escrow reservation
   - Authorizes mailbox slot
4. Maker sets hashlock via AtomicDesk.setHashlock()
5. Taker sends encrypted Monero context via Mailbox
6. Maker responds with encrypted presignature via Mailbox
7. Taker completes signature, broadcasts Monero tx
8. Maker settles EVM side via SettlementEscrow.settle(tau)
   - Verifies keccak256(tau) == hashlock
   - Transfers collateral to taker
```

### 10.6 Security Properties

- **Atomicity**: Both parties receive assets or neither does
- **Privacy**: Monero details remain private (encrypted in mailbox)
- **Trustlessness**: No intermediaries required
- **Collateralization**: All EVM assets fully backed in DeskVault
- **Committee Oversight**: Refund path for stuck reservations

---

## 11. Security Model

### 11.1 Core Invariants

#### DeskVault Invariants
1. `free + reserved + fees = actual balance` (per token, per desk)
2. Reserved balances never negative
3. Reserved balances only change via authorized calls
4. Makers cannot withdraw reserved assets

#### AuctionHouse Invariants
1. `remainingVolume <= maxVolume`
2. `remainingVolume` strictly decreases on fills
3. Auctions cannot change price after creation
4. All auctions fully collateralized

#### Router Invariants
1. No external calls between state transitions
2. User constraints enforced before transfers
3. Atomic settlement (all-or-nothing)

### 11.2 Threat Model

#### Malicious Executors Cannot:
- Change price (deterministic curve)
- Violate user constraints
- Alter fee asset
- Extract hidden fees
- Replay orders

#### Malicious Makers Cannot:
- Withdraw reserved liquidity
- Modify auction parameters
- Manipulate settlement price
- Force settlement
- Rug users

#### Malicious Swappers Cannot:
- Cause partial fills
- Manipulate state
- Replay orders (nonce protection)

### 11.3 MEV Resistance

EqualX eliminates MEV vectors:
- **No Sandwiching**: Price predetermined
- **No Backrunning**: Atomic settlement
- **No Oracle Manipulation**: Maker-set prices
- **No Solver Privilege**: Open execution
- **No Pathfinding Arbitrage**: Single-curve fills

---

## 12. Off-Chain Components

### 12.1 Rust Crates

| Crate | Purpose |
|-------|---------|
| **adaptor-clsag** | CLSAG adaptor signatures |
| **equalx-sdk** | High-level SDK with Alloy integration |
| **presig-envelope** | Encryption/decryption |
| **monero-wallet-core** | Monero integration |
| **monero-rpc** | RPC client |
| **tx_builder** | Transaction construction |
| **watcher** | Event monitoring |
| **cli** | Command-line interface |

### 12.2 FFI Bindings

| Crate | Target |
|-------|--------|
| **ffi-c** | C/C++ integration |
| **ffi-wasm** | WebAssembly/browser |

### 12.3 Tools

| Tool | Purpose |
|------|---------|
| **build_swap_tx** | Construct Monero transactions |
| **export_tx** | Export transaction data |
| **send_tx** | Broadcast to Monero network |

---

## 13. Gas Efficiency

### 13.1 Typical Costs

| Operation | Gas | Notes |
|-----------|-----|-------|
| ERC20 → ERC20 swap | ~150-200k | Direct execution |
| ETH → ERC20 swap | ~130-170k | Dedicated ETH entry |
| Create curve | ~650k | Includes commitment |
| Update curve | ~110-120k | Adaptive update |
| Cancel curve | ~640k | Returns collateral |
| Option exercise | ~200k | Includes burn + settle |
| Atomic reservation | ~800k-1.1M | Includes escrow |

### 13.2 Optimizations

- Minimal onchain state (commitments, not full descriptors)
- Batch operations for curves
- Efficient storage layout
- No unnecessary external calls
- Direct ETH transfers (no wrapping)

---

## 14. Composability

### 14.1 Composability Model

EqualX enables:
- **Direct trading UIs**: Users execute swaps directly
- **Automated strategies**: Offchain curve creation
- **Derivative layers**: Options, futures, atomic swaps
- **Vertical liquidity**: Project-specific desks

Without:
- Centralizing liquidity
- Introducing privilege
- Weakening guarantees
- Limiting access

### 14.2 Extension Points

1. **New Desk Types**: LaunchDesk, OptionsDesk, FuturesDesk, AtomicDesk
2. **New Collateral Types**: ERC20, ETH, derivatives
3. **Cross-chain**: Atomic desks, adaptor signatures

### 14.3 Fork Resistance

EqualX's structure is inherently fork-resistant:
- No protocol-level fee capture
- No governance knobs to modify
- Deterministic core logic
- Sovereign Desks for differentiation
- UIs and strategy live offchain
- Curves globally accessible despite siloed liquidity

---

## Appendices

### A. Contract Addresses

*Deployment addresses to be added after mainnet launch.*

### B. Deployment Order

```solidity
1. DeskVault
2. AuctionHouse
3. Router
4. AmmAuctionManager
5. SettlementEscrow
6. AtomicDesk
7. Mailbox
8. EncPubRegistry
9. OptionsToken & OptionsManager
10. FuturesLongToken & FuturesManager
```

### C. Configuration

```solidity
// Wire core contracts
vault.configureAuctionHouse(auctionHouse);
vault.configureRouter(router);
vault.configureSettlementEscrow(escrow);
vault.configureAtomicDeskController(atomicDesk);

// Configure derivatives
vault.setTrustedDeskAgent(optionsManager, true);
vault.setTrustedDeskAgent(futuresManager, true);

// Configure AMM
vault.configureAmmAuctionManager(ammManager);

// Configure atomic
escrow.configureMailbox(mailbox);
escrow.configureAtomicDesk(atomicDesk);
```

### D. Document References

#### Core Documentation
- `docs/MAM.md` — Maker Auction Markets primer
- `docs/DESKVAULT.md` — Collateral management
- `docs/AUCTIONHOUSE.md` — Curve registry
- `docs/ROUTER-DESIGN.md` — Settlement engine
- `docs/FEEMODEL.md` — Fee mechanics

#### Specialized Features
- `docs/ATOMICDESKS-DESIGN.md` — Cross-chain swaps
- `docs/OPTIONSDESK.md` — Options layer
- `docs/FUTURESDESK.md` — Futures layer
- `docs/LAUNCHDESK.md` — Token launches
- `docs/MAILBOX-DESIGN.md` — Encrypted communication

#### Security & Design
- `docs/SECURITYMODEL.md` — Threat analysis
- `docs/COMPOSABILITY.md` — Extension model
- `docs/EXECUTIONMODES.md` — Intent handling

#### Cryptographic Specifications
- `docs/CLSAG-ADAPTOR-SPEC.md` — CLSAG adaptor signatures
- `docs/FCMP-ADAPTOR-SPEC.md` — FCMP adaptor signatures

#### Planning & Status
- `PLAN.md` — Router/Vault simplification roadmap
- `TASKS.md` — Hash consistency tasks
- `DIVERGE.md` — Spec vs implementation divergences
- `GAS-ESTIMATES.md` — Gas efficiency report

### E. Implementation Notes & Simplifications

The current implementation has been **simplified** from earlier designs documented in some `.md` files:

#### Router Simplifications (per PLAN.md)
1. **Removed delegated/trusted execution** - Only direct execution remains
2. **Removed EIP-712 signatures, nonces, tips** - No meta-transaction support
3. **Removed deskScope** - No desk-scoped intents
4. **Removed slippageBps/referencePriceX18** - Use `minAmountOut` for protection
5. **TokenIn fees only** - TokenOut fee path removed
6. **No support donations** - `supportBps` and `supportAddress` must be 0
7. **Dedicated ETH entry** - `executeDirectEth` for ETH paths

#### DirectIntent Minimized
```solidity
// Current (minimal)
struct DirectIntent {
    uint256 curveId;
    uint128 amountIn;
    uint128 minAmountOut;
    uint64 userDeadline;
}

// Earlier designs included (now removed):
// - user address
// - slippageBps
// - referencePriceX18
// - deskScope
// - feeAssetAllowed
```

#### Curve Validation Simplified
- Router loads curve data via `loadCurveForFill(curveId)`
- No calldata descriptor verification against stored commitment
- Commitment verification happens at curve creation/update time

#### Documentation vs Code Discrepancies
Some documentation files describe features not in the current implementation:
- `docs/EXECUTIONMODES.md` - Describes delegated execution (not implemented)
- `docs/ROUTER-DESIGN.md` - Describes more complex Router (simplified)
- `docs/FEEMODEL.md` - Describes TokenOut fees and tips (removed)

**Always refer to the smart contract source code as the source of truth.**

### F. Known Issues & Divergences

#### CLSAG Adaptor Specification (per DIVERGE.md)
Current implementation diverges from v1 spec:
1. Transcript namespace uses `v0.0.1` instead of `v1`
2. Pre-hash uses SHA3-256 instead of BLAKE2b
3. Binary wire formats differ from spec JSON structures
4. `board_id` and `chain_tag` included in transcript (spec says removed)

**Note on τ:** The spec's language about τ being "secret until final signature" is misleading. τ is not a secret—it is a **proof of Monero spend**. The taker can extract τ as soon as they construct the final transaction (even before broadcast). τ's purpose is to prove to the Maker (or committee) that the XMR spend occurred. The deterministic τ derivation in the code is correct; the spec language should be clarified.

**Recommendation:** Align implementation with v1 spec or update spec to match implementation.

#### Hash Consistency (per TASKS.md)
Tasks to standardize on SHA3-256:
- Pre-hash computation
- Ring/message/settlement hashing
- Designated index derivation

#### Planned
1. FCMP Integration (Full-Chain Membership Proofs)
2. Multi-Asset Atomic Swaps
3. Advanced Key Management
4. Cross-Chain Extensions (Bitcoin, other UTXO chains)
5. Batch Operations

#### Potential
1. Margin-backed Writers (partial collateralization)
2. Volatility-adjusted Premiums
3. Secondary Markets for derivatives
4. Exotic Structures (barriers, collars, binaries)
5. Layer 2 Optimization

---

## Conclusion

EqualX represents a fundamental shift in DEX architecture:

**From reactive to deterministic** — Prices set by Makers, not pools

**From pooled to sovereign** — Each Maker controls their own inventory

**From privileged to permissionless** — Any executor can settle

**From complex to simple** — Single-curve settlement, minimal state

The result is a protocol that provides user safety, maker sovereignty, MEV resistance, accessibility, and composability — enabling a new class of trading applications where users can trust that their signed intents will be executed exactly as specified, while makers can provide liquidity with complete predictability and control.

---

*This document consolidates information from all EqualX documentation, smart contracts, and off-chain components into a single comprehensive reference.*


### G. Future Enhancements

#### Planned
1. FCMP Integration (Full-Chain Membership Proofs)
2. Multi-Asset Atomic Swaps
3. Advanced Key Management
4. Cross-Chain Extensions (Bitcoin, other UTXO chains)
5. Batch Operations

#### Potential
1. Margin-backed Writers (partial collateralization)
2. Volatility-adjusted Premiums
3. Secondary Markets for derivatives
4. Exotic Structures (barriers, collars, binaries)
5. Layer 2 Optimization

---

## Conclusion

EqualX represents a fundamental shift in DEX architecture:

**From reactive to deterministic** — Prices set by Makers, not pools

**From pooled to sovereign** — Each Maker controls their own inventory

**From privileged to permissionless** — Any user can execute directly

**From complex to simple** — Single-curve settlement, minimal state, direct execution only

The result is a protocol that provides user safety, maker sovereignty, MEV resistance, accessibility, and composability — enabling a new class of trading applications where users can trust that their constraints will be enforced exactly as specified, while makers can provide liquidity with complete predictability and control.

---

*This document reflects the current implementation as verified against the smart contract source code in `contracts/`. Some older documentation files may describe features that have been simplified or removed. Always refer to the source code as the authoritative reference.*
