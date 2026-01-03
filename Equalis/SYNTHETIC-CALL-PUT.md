# Synthetic Calls and Puts

The Equal Lend **Direct** rail can express "call" and "put" payoffs because the borrower's two terminal choices map cleanly to **exercise vs not exercise**:

* **Repay** the borrow asset and unlock collateral = "don't exercise"
* **Default / Exercise** (intentional early settle, if enabled) and forfeit locked collateral = "exercise" 

Direct is explicitly designed so that this "option-like" behavior comes from the **repay vs collateral-seizure** branch, not from any oracle-based pricing. 

## Synthetic call (buyer wants upside on ETH)

Set the deal so the borrower receives the underlying and posts stable collateral:

* `borrowAsset = ETH` (or rETH)
* `principal = Q ETH`
* `collateralAsset = USDC` (via the borrower's collateral pool)
* `collateralLockAmount = K * Q USDC` (this is the "strike * size" chosen by the lender/offer) 
* `userInterest` (paid upfront) is the **premium** the buyer pays to the writer (plus any platform fees) 
* `dueTimestamp = T` expiry 

Payoff intuition:

* If ETH ends **above** K: borrower repays Q ETH, gets KQ USDC back (economically like exercising a call).
* If ETH ends **below** K: borrower defaults/exercises, keeps the ETH, forfeits KQ USDC (economically like letting the call expire worthless from the buyer's view, but note: this is a *non-recourse* framing).

European vs American:

* If exercise can only happen at/after due timestamp, it's European-style.
* If `allowEarlyExercise` is enabled, borrower can "exercise" early (American-style) without oracles. 

## Synthetic put (buyer wants downside protection on ETH)

Flip which asset is borrowed vs posted as collateral:

* `borrowAsset = USDC`
* `principal = K * Q USDC`
* `collateralAsset = ETH`
* `collateralLockAmount = Q ETH` 
* `userInterest` upfront is the put premium 

Payoff intuition:

* If ETH ends **below** K: borrower defaults/exercises, keeps KQ USDC, forfeits Q ETH (economically "sold ETH at K," i.e., put exercised).
* If ETH ends **above** K: borrower repays KQ USDC, unlocks their Q ETH (put expires worthless).

## Why this works without oracles

The "strike" is not computed onchain. It's implied by the fixed quantities the offer sets: `(principal, collateralLockAmount)` and expiry. There's no onchain LTV check or price conversion for Direct; the lender chooses collateral size explicitly.

## Implementation notes

- **Exercise vs Default**: The code distinguishes `exerciseDirect()` (borrower-initiated, sets status `Exercised`) from `recover()` (anyone can call after grace period, sets status `Defaulted`). Both transfer collateral to the lender.
- **Grace period**: 1-day grace period after `dueTimestamp`. Repay/exercise must happen within this window; `recover()` becomes available after it expires.
- **`allowEarlyRepay`**: Separate from `allowEarlyExercise`. When false, borrower can only repay within the grace window (not before `dueTimestamp - 1 day`).
- **`allowLenderCall`**: Lets the lender accelerate `dueTimestamp` to the current block, forcing the borrower into the repay-or-exercise decision early.
