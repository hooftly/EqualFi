# Yield-Bearing Limit Orders (YBLOs)

Yield-Bearing Limit Orders (YBLOs) are Equalis-native limit orders that let you post standing buy or sell walls while keeping your capital inside an Equalis pool. Your funds are not withdrawn to an external order book contract or parked in a dead escrow. Instead, they remain in-pool and are represented as an onchain encumbrance.

That encumbrance makes the funds non-withdrawable and non-reusable, but (depending on the pool’s accounting rules) the encumbered balance can continue to accrue in the same way as other pool balances. This is the “yield-bearing” part.

This document explains what YBLOs are, how they work, and what tradeoffs they introduce.

## What problem YBLOs solve

Traditional onchain limit orders usually require you to move assets into a dedicated order contract that does not generate yield. Your capital sits idle while you wait for a fill.

YBLOs aim to make “waiting” less wasteful by letting deposits stay in the pool while still being safely committed to an executable order.

## The core idea

A YBLO is a standing offer:

- Sell: “I will sell up to X of token A for token B at price P or better”
- Buy: “I will buy up to X of token A using token B at price P or better”

Instead of transferring X out of the pool, Equalis:

1) Encumbers X inside the maker’s pool position
2) Publishes the order parameters onchain (events plus order state)
3) Allows any taker to fill the order (fully or partially), subject to its limits
4) Settles the spot transfer between maker and taker positions

At no point does the maker need to be online for the order to remain available.

## How it works (lifecycle)

### 1) Create
You deposit into an Equalis pool (or already have a Position with balance). You create a YBLO specifying:

- Sell token / buy token
- Amount offered (or remaining)
- Limit ratio (X of Token A for Y of Token B)

Equalis marks the posted amount as encumbered.

### 2) Wait
While open:

- Encumbered funds cannot be withdrawn
- Encumbered funds cannot be reused for another obligation
- Encumbered funds can remain in the pool accounting domain

Whether the encumbered portion “earns yield” depends on the pool’s rules. The design intent is that it can, but you should treat this as pool-defined behavior, not magic.

### 3) Fill (spot trade)
A taker fills some amount.

Settlement happens directly between positions:

- Maker’s encumbered asset decreases by filled amount
- Maker receives the counter-asset
- Taker pays the counter-asset and receives the asset

Partial fills are supported if the order allows it.

### 4) Close
The order ends by:

- Fully filled
- Cancelled by maker (remaining encumbrance is released)

## What makes YBLOs different from normal limit orders

### “Yield-bearing” is not marketing fluff
The difference is not a higher APY promise. It is mechanical:

- Your order funds stay inside an Equalis pool
- Equalis enforces commitment through encumbrances
- Pool accounting may keep accruing on encumbered funds

In contrast, most limit orders move funds into a separate escrow where yield is zero unless additional complexity is added.

### Encumbrances make commitments explicit
YBLOs use the same “explicit obligation” philosophy used across Equalis:

- Your balance sheet is verifiable onchain
- Commitments are enforceable without trusting offchain actors
- Double-spend of the same deposit is prevented by construction

## What YBLOs are not

- Not an AMM.
  - There is no continuously moving curve and no routing.
  - Execution is against your stated terms, not against a pool price.
- Not a promise of best execution.
  - It is a standing offer. If your price is bad, you get bad fills.
- Not a promise of yield in all pool configurations (yield behavior depends on pool accounting rules).

This prevents misinterpretation and makes the doc more defensible.

## Encumbrance clarity

Encumbered means non-withdrawable and non-reusable, enforced onchain.

## Gas costs

YBLOs are onchain objects with explicit encumbrance accounting and per-position state updates. That design makes commitments verifiable and enforceable without offchain trust, but it also means the core operations are storage-heavy and therefore not “ultra-cheap swaps.”

To be concrete, here are the measured gas costs from Foundry tests (`DirectLimitOrderGas.t.sol`) for the current implementation:

* **Post YBLO:** 549,866 (no fees), 548,086 (fees ratio)
* **Accept YBLO:** 347,048 (no fees), 418,590 (fees ratio)
* **Accept borrower-side:** 418,792 (fees ratio)
* **Cancel YBLO:** 57,805 (no fees), 58,091 (fees ratio)

A few important implications:

* The dominant cost is **posting**, which is ~550k gas.
* **Acceptance** is materially cheaper than posting, landing around **~419k** gas.
* **Cancellation** is cheap relative to the other actions, around **~58k** gas.

These numbers reflect the current architecture and are not presented as “best-in-class.” They are the cost of making obligations explicit (encumbrances), keeping state machine transitions onchain, and preserving correctness without relying on privileged matching infrastructure.


## Why this matters

### 1) Standing walls that do not idle capital
Market makers and users can post buy and sell walls without fully giving up productive use of capital inside the protocol domain.

### 2) Cleaner agent automation
For autonomous agents, predictable settlement beats reactive liquidation mechanics. A YBLO is a deterministic object:

- It is either fillable under defined terms, or not
- A fill results in a defined state update and transfer

### 3) Composability with credit primitives
Equalis supports both credit and trade. YBLOs are the trade-facing surface that shares the same commitment primitives. This keeps the system consistent rather than bolting a DEX onto a lending protocol.

## Risks and tradeoffs

- Smart contract risk: bugs beat philosophy.
- Token risk: non-standard ERC20 behavior can cause edge cases.
- Price risk: you are posting a limit order. Adverse selection is real.
- Liquidity risk: your funds are encumbered. You cannot instantly pull them if the market shifts unless you cancel (which costs gas).
- Gas risk: fills are currently expensive relative to minimalist swap paths.

## Simple example

You want to buy ETH with USDC.

1) Deposit USDC into the relevant pool position.
2) Create a YBLO: “Spend up to 10,000 USDC to buy ETH at <= 2,400 USDC/ETH.”
3) Your 10,000 USDC becomes encumbered.
4) A taker sells you ETH into your order at 2,400 or better.
5) You receive ETH, the taker receives USDC, and any remaining encumbrance stays open until filled or cancelled.

## FAQ

### Does my posted amount always earn yield?
Not automatically. The protocol design allows encumbered balances to remain in the pool accounting domain, but the exact yield behavior is pool-defined. The correct mental model is: encumbered does not mean removed, but it does mean restricted.

### Can I partially fill and keep the rest open?
Yes, if the order supports partial fills. The remaining amount stays encumbered and available.

### Can I cancel?
Yes. Cancellation releases the remaining encumbrance. It costs gas.

### Is this better than using a DEX?
Sometimes. YBLOs are not trying to beat AMMs on raw swap efficiency today. They are trying to enable standing liquidity that does not force capital to sit idle and that fits the non-reactive, explicit-commitment design of Equalis.

## Summary

YBLOs are limit orders funded by pool balances, enforced by explicit encumbrances, and settled as spot trades between positions. They are designed to let users and market makers post executable onchain liquidity while keeping capital inside the Equalis accounting domain.

