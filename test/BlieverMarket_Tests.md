# BlieverMarket — Test Suite Documentation

> **For:** Test developers, protocol engineers, auditors, and anyone debugging or extending BlieverMarket tests.  
> **Companion contract:** `contracts/src/BlieverMarket.sol`  
> **Test folder:** `contracts/test/BlieverMarket/`  
> **Stack:** Solidity 0.8.31 · Foundry · Base Chain (Cancun EVM)

---

## Quick Reference

| File | Test Contract | What it covers |
|---|---|---|
| `BlieverMarketBase.t.sol` | `BlieverMarketBase` *(abstract)* | Shared mocks, fixtures, helpers |
| `BlieverMarket_Init.t.sol` | `BlieverMarket_InitTest` | initialize() — all branches |
| `BlieverMarket_Trading.t.sol` | `BlieverMarket_TradingTest` | buy(), sell(), CSS |
| `BlieverMarket_Resolution.t.sol` | `BlieverMarket_ResolutionTest` | resolve(), claim(), expireUnresolved() |
| `BlieverMarket_Admin.t.sol` | `BlieverMarket_AdminTest` | pause/unpause, all view functions |
| `BlieverMarket_Fuzz.t.sol` | `BlieverMarket_FuzzTest` | Property-based fuzz tests |
| `BlieverMarket_Invariant.t.sol` | `BlieverMarket_InvariantTest` | Stateful invariant fuzzing (Handler pattern) |
| `BlieverMarket_Gas.t.sol` | `BlieverMarket_GasTest` | Gas profiling and regression caps |

**Run all tests:**
```bash
forge test --match-path "test/BlieverMarket/**" --evm-version cancun -vvv
```

**Run a specific file:**
```bash
forge test --match-contract BlieverMarket_TradingTest --evm-version cancun -vvv
```

**Gas snapshot:**
```bash
forge snapshot --evm-version cancun
forge snapshot --check --evm-version cancun   # CI regression gate
```

> **EVM version note:** `--evm-version cancun` is required because `BlieverMarket` uses `ReentrancyGuardTransient` which depends on EIP-1153 transient storage (TSTORE/TLOAD), introduced in Cancun.

---

## Test Parameters

All tests share these market parameters, chosen to match a realistic production deployment:

| Parameter | Value | Meaning |
|---|---|---|
| `ALPHA` | `3e16` (3%) | LS-LMSR commission parameter α |
| `MAX_RISK_USDC` | `500e6` | Pool risk budget per market: 500 USDC |
| `EPSILON_2` | `480e18` | Seed quantity for 2-outcome market: ε ≈ R / (1 + α·2·ln2) |
| `EPSILON_7` | `355e18` | Seed quantity for 7-outcome market: ε ≈ R / (1 + α·7·ln7) |
| `TRADER_USDC` | `1_000e6` | Default USDC balance given to each test trader |
| `POOL_SEED` | `100_000e6` | USDC pre-funded in MockPool for refunds/claims |
| `MIN_SHARE` | `1e15` | BlieverMarket.MIN_SHARE_AMOUNT (dust guard) |

Two live market clones are deployed in `setUp()`:
- **`market2`** — 2-outcome (canonical binary prediction market)
- **`market7`** — 7-outcome (multi-outcome / CSS stress scenarios)

---

## File-by-File Breakdown

---

### `BlieverMarketBase.t.sol` — Shared Infrastructure

**Role:** Abstract base inherited by all test contracts. Not collected as a test suite by Foundry.

**Mock contracts defined here:**

| Contract | Purpose |
|---|---|
| `MockUSDC` | 6-decimal ERC-20 stub. `mint()`, `approve()`, `transfer()`, `transferFrom()`. No EIP-2612 permit (permit tests pass `v=0`). |
| `MockPool` | Implements `IBlieverV1Pool`. Executes real USDC transfers so token-balance assertions work end-to-end. Exposes spy counters (`collectCalls`, `refundCalls`, `settleCalls`, `claimCalls`) and last-call arguments (`lastTrader`, `lastCost`, etc.) for assertion convenience. `seed()` pre-funds the pool. `resetCounters()` resets counters between assertions in a single test. |

**Helpers:**

| Helper | What it does |
|---|---|
| `_setupTrader(addr, amount)` | Mints USDC to trader and approves pool with `type(uint256).max`. Mirrors the exact on-chain UX flow. |
| `_buy(market, trader, outcomeIdx, shareAmt)` | Calls `getBuyCost()`, mints enough USDC, then calls `market.buy()` with 2× cost cap. Returns actual cost. |
| `_sell(market, trader, outcomeIdx, shareAmt)` | Calls `getSellEstimate()` then `market.sell()` with `minRefund=0`. Returns estimated refund. |
| `_resolve(market, winner)` | Pranks resolver and calls `resolve(winner)`. |
| `_newClone(nOutcomes, epsilon, qId)` | Deploys a fresh EIP-1167 clone from the shared `impl`. Useful for isolated tests that don't want side effects from `market2`/`market7`. |
| `_sharesSlot(trader, idx)` | Computes the `keccak256` storage slot for `_shares[trader][idx]`. Used in the dust test to inject sub-`MIN_SHARE` balances directly. |
| `_claimedSlot(trader)` | Computes the storage slot for `_claimed[trader]`. |

---

### `BlieverMarket_Init.t.sol` — Initialization Tests

**Run:** `forge test --match-contract BlieverMarket_InitTest`

**What is tested:**

#### Happy Path — Storage State
- All config variables (`pool`, `outcomeCount`, `resolved`, `resolver`, `tradingDeadline`, `resolutionDeadline`, `factory`, `questionId`, `alpha`, `usdc`) are written correctly for both 2-outcome and 7-outcome markets.
- Initial quantity vector `q⁰ = [ε, ε, …, ε]` is correct length and uniform.
- `getInitialQuantities()` matches `getQuantities()` before any trades.
- `usdc` slot is populated from `pool.asset()` at init (no live call in trading hot path).
- Initial prices are symmetric (equal for all outcomes at uniform seed).

#### Happy Path — Event Emission
- `MarketInitialized` event is emitted with all correct indexed and non-indexed parameters.
- Market begins in `tradingOpen = true` state immediately after initialization.

#### Revert — Zero Address Inputs
- `_pool == address(0)` → `ZeroAddress`
- `_resolver == address(0)` → `ZeroAddress`
- `_factory == address(0)` → `ZeroAddress`

#### Revert — Outcome Count
- `outcomeCount = 0` → `InvalidOutcomeCount(0)`
- `outcomeCount = 1` → `InvalidOutcomeCount(1)`
- `outcomeCount = 101` → `InvalidOutcomeCount(101)`

#### Revert — Alpha
- `alpha < 1e12` (below `MIN_ALPHA`) → `InvalidAlpha`
- `alpha > 2e17` (above `MAX_ALPHA`) → `InvalidAlpha`

#### Revert — Epsilon / Deadlines
- `epsilon = 0` → `ZeroEpsilon`
- `tradingDeadline == block.timestamp` → `InvalidDeadlines` (must be strictly future)
- `tradingDeadline` in the past → `InvalidDeadlines`
- `resolutionDeadline == tradingDeadline` → `InvalidDeadlines` (must be strictly after)
- `resolutionDeadline < tradingDeadline` → `InvalidDeadlines`

#### Double-Init / Master Lock
- Second `initialize()` call on same clone → reverts (OZ `InvalidInitialization`)
- `initialize()` on the master implementation → reverts (`_disableInitializers()` in constructor)

#### Boundary — Valid Extremes
- `outcomeCount = 2` (minimum) accepted
- `outcomeCount = 100` (maximum) accepted
- `alpha = 1e12` (MIN_ALPHA) accepted
- `alpha = 2e17` (MAX_ALPHA) accepted

---

### `BlieverMarket_Trading.t.sol` — Buy + Sell + CSS Tests

**Run:** `forge test --match-contract BlieverMarket_TradingTest`

**What is tested:**

#### Buy — Happy Path
- Shares credited to `_shares[alice][outcomeIndex]` after buy.
- Quantity vector: only the purchased outcome's slot incremented; others unchanged.
- Buy works for every valid outcome index on a 7-outcome market.
- Multiple buys by the same trader accumulate correctly.
- `_totalTraderShares[outcomeIndex]` tracks the aggregate position across traders.
- `pool.collectTradeCost()` is called exactly once with the correct trader address and non-zero cost.
- Actual cost matches `getBuyCost()` quote within ±1 USDC-wei (ceiling rounding).
- `Bought` event emitted with all correct arguments.
- Price of bought outcome increases after trade (AMM monotonicity).
- Price of non-bought outcome decreases after trade (LS-LMSR property).
- Trader's USDC balance decreases by the cost paid.

#### Buy — Reverts
- `shareAmount < MIN_SHARE_AMOUNT` → `ShareAmountTooSmall`
- `outcomeIndex >= outcomeCount` → `InvalidOutcomeIndex`
- `maxCostUsdc = 0` (actual cost > 0) → `SlippageExceeded`
- While paused → `EnforcedPause`
- After `tradingDeadline` → `TradingClosed`
- After resolution → `TradingClosed`
- At exactly `tradingDeadline` (boundary: `block.timestamp == tradingDeadline` is NOT allowed) → `TradingClosed`

#### Sell — Standard Refund Path
- Selling held shares: share balance decreases, USDC refunded to trader.
- Selling all held shares zeroes the balance.
- Quantity vector: sold outcome's slot decrements, others unchanged.
- `pool.distributeRefund()` called once with correct trader.
- `Sold` event emitted with `cssTranslation = 0` on a standard sell.
- Round-trip cost ≥ refund (AMM spread always positive; traders cannot profit on buy+sell).

#### Sell — CSS (Covered Short Selling)
- **Zero holdings CSS**: Selling outcome 0 with 0 shares → `tBar = shareAmount`, `netReduce = 0`. Market: `q[0]` unchanged, `q[1] += tBar`. Trader: `shares[0] = 0`, `shares[1] = tBar`.
- **Partial holdings CSS**: Holding 1 share, selling 3 → `tBar = 2`, `netReduce = 1`. Verified quantity deltas and share ledger updates.
- **7-outcome CSS**: Selling outcome 0 with zero holdings → `tBar` distributed to all 6 other outcomes in both quantity vector and trader share ledger.
- `getCssTranslation()` returns correct `tBar` before and after buys.

#### Sell — Reverts
- `shareAmount < MIN_SHARE_AMOUNT` → `ShareAmountTooSmall`
- `outcomeIndex >= outcomeCount` → `InvalidOutcomeIndex`
- `minRefundUsdc > actual refund` → `SlippageExceeded`
- While paused → `EnforcedPause`
- After `tradingDeadline` → `TradingClosed`
- After resolution → `TradingClosed`

#### View Estimates
- `getSellEstimate()` output matches actual pool refund within ±1 USDC-wei.
- `getSellEstimate(trader, idx, 0)` returns `(0, 0)`.
- `getBuyCost(idx, 0)` returns `0`.

---

### `BlieverMarket_Resolution.t.sol` — Resolve / Claim / Expire Tests

**Run:** `forge test --match-contract BlieverMarket_ResolutionTest`

**What is tested:**

#### Resolve — Happy Path
- `resolved = true`, `winningOutcome` stored, `pool.settleMarket()` called exactly once.
- `settleMarket(0)` when no traders have bought (epsilon seed excluded from payout).
- `settleMarket(totalPayout)` where `totalPayout = floor(totalTraderShares[winner] / 1e12)`.
- `MarketResolved` event emitted with correct indexed/data fields.
- After resolve, `buy()` and `sell()` revert with `TradingClosed`.
- Resolve is NOT pause-gated (market is always settleable).

#### Resolve — Reverts
- Non-resolver caller → `NotResolver`
- Second call → `MarketAlreadyResolved`
- After `resolutionDeadline` → `ResolutionDeadlinePassed`
- At exactly `resolutionDeadline` → `ResolutionDeadlinePassed`
- `winningOutcome >= outcomeCount` → `InvalidOutcomeIndex`

#### Claim — Happy Path
- Winner receives `floor(shares18 / 1e12)` USDC from pool.
- `WinningsClaimed` event emitted with correct arguments.
- `pool.claimWinnings()` called with winner's address and correct amount.
- Multiple winners claim independently with correct proportional payouts.
- Loser (holding non-winning outcome shares) cannot claim (`NoWinningShares`).
- Claim is NOT pause-gated.
- **Dust path**: shares < `1e12` (converts to 0 USDC) → `DustForfeited` event emitted, `_claimed[caller] = true`, `pool.claimWinnings` is NOT called.

#### Claim — Reverts
- Before resolution → `MarketNotResolved`
- Second claim by same address → `AlreadyClaimed`
- Trader has zero winning shares → `NoWinningShares`
- Trader never bought winning outcome → `NoWinningShares`

#### Expire Unresolved — Happy Path
- Sets `resolved = true`, calls `pool.settleMarket(0)`.
- `winningOutcome` set to `outcomeCount` (out-of-range index prevents any claims).
- `MarketExpired` event emitted with factory address, timestamp, and `ExpiryReason.TIMEOUT`.
- Expire is NOT pause-gated.
- After expiry, all traders with positions get `NoWinningShares` on `claim()` (since no one holds shares at `winningOutcome = outcomeCount`).

#### Expire Unresolved — Reverts
- Non-factory caller → `NotFactory`
- Market already resolved → `MarketAlreadyResolved`
- Before `resolutionDeadline` → `ResolutionDeadlineNotPassed`
- At exactly `resolutionDeadline` → `ResolutionDeadlineNotPassed`

#### Combined Lifecycle
- End-to-end: buy → resolve → claim for winner; loser cannot claim.
- `getMarketStatus()` correctly reflects state at each lifecycle stage.

---

### `BlieverMarket_Admin.t.sol` — Admin & View Tests

**Run:** `forge test --match-contract BlieverMarket_AdminTest`

**What is tested:**

#### Admin — Pause / Unpause
- `pause()` by factory: `buy()` and `sell()` revert with `EnforcedPause`.
- `pause()` by non-factory (attacker, resolver) → `NotFactory`.
- `unpause()` by factory: trading resumes normally.
- `unpause()` by non-factory → `NotFactory`.
- Both `buy()` and `sell()` are blocked while paused.

#### Asymmetric Pause Design
These functions must succeed even while the market is paused (mirrors pool design):
- `resolve()` — always settleable
- `claim()` — always claimable  
- `expireUnresolved()` — always expirable

#### Views — Prices
- At initialization, all prices are symmetric (equal) for 2-outcome market.
- `getAllPrices()` returns arrays of length `outcomeCount` for both 2- and 7-outcome markets.
- `getAllPrices()` values match individual `getPrice()` calls.
- `getSumOfPrices()` > 1e18 before and after trades (LS-LMSR spread property).
- `getPrice()` reverts for out-of-range outcome index.

#### Views — Estimates
- `getBuyCost()` returns positive value for valid share amounts.
- `getBuyCost()` is monotonically increasing (larger amount → higher cost).
- `getSellEstimate()` returns `(refund > 0, cost = 0)` after buying.
- `getSellEstimate()` on CSS path (zero holdings): at most one of `refund`/`cost` is non-zero.

#### Views — Shares & Quantities
- `getShares()` returns 0 before any trade; correct after buy.
- `getAllShares()` returns length-`outcomeCount` array matching individual `getShares()`.
- `getShares()` reverts for out-of-range index.
- `getQuantities()` returns `[ε, ε]` at initialization; updates after buy.
- `getInitialQuantities()` never changes (immutable after `initialize()`).
- `getTotalTraderShares()` excludes epsilon seed (returns 0 before any trades).
- `getTotalTraderShares()` correctly aggregates across multiple traders.
- `getTotalTraderShares()` reverts for out-of-range index.

#### Views — Market Status & Misc
- `getMarketStatus()` returns correct values at initialization (all fields verified).
- `totalVolumeShares` in `getMarketStatus()` increases by `shareAmount` after each buy.
- `tradingOpen = false` after resolution.
- `usdcToken()` returns the cached USDC address.
- `hasClaimed()` is `false` before claim, `true` after.
- `getCssTranslation()` returns 0 for out-of-range outcome index (no revert, defensive).

---

### `BlieverMarket_Fuzz.t.sol` — Fuzz Tests

**Run:** `forge test --match-contract BlieverMarket_FuzzTest`

**Configure in `foundry.toml`:**
```toml
[fuzz]
runs = 2000
seed = "0xBELIEVER"
```

**Properties tested:**

| Test | Property |
|---|---|
| `testFuzz_getBuyCost_positiveForAnyValidAmount` | `getBuyCost(i, amt) > 0` for all `amt ≥ MIN_SHARE` |
| `testFuzz_getBuyCost_monotoneIncreasing` | `amt1 < amt2 → cost(amt1) ≤ cost(amt2)` |
| `testFuzz_buy_priceIncreases` | Buying any amount increases the outcome's price |
| `testFuzz_buy_otherOutcomePricesDecrease` | Buying outcome 0 decreases outcome 1's price |
| `testFuzz_buy_sumOfPricesAlwaysGtOne` | `Σ prices > 1` after any buy (LS-LMSR spread) |
| `testFuzz_roundTrip_refundLEcost` | Sell refund ≤ buy cost for any amount (AMM spread) |
| `testFuzz_sell_css_sharesNeverNegative` | CSS leaves `shares[trader][i] ≥ 0` for all i |
| `testFuzz_sell_quantityNeverBelowEpsilon` | `q[i] ≥ 0` after any buy + sell sequence |
| `testFuzz_cssTranslation_consistent` | `getCssTranslation()` output matches expected `tBar` formula |
| `testFuzz_buy_alwaysReverts_zeroMaxCost` | `buy(..., maxCost=0)` always reverts |
| `testFuzz_sell_zeroMinRefund_neverReverts` | `sell(..., minRefund=0)` never false-reverts |
| `testFuzz_buy_succeedsWhenCapGeActualCost` | `buy()` always succeeds when `maxCost ≥ actualCost` |
| `testFuzz_buy_revert_belowMinShare` | Any `amt < 1e15` reverts with `ShareAmountTooSmall` |
| `testFuzz_sell_revert_belowMinShare` | Any `amt < 1e15` reverts with `ShareAmountTooSmall` |
| `testFuzz_buy_revert_invalidOutcomeIndex` | Any `idx ≥ outcomeCount` reverts with `InvalidOutcomeIndex` |
| `testFuzz_sell_revert_invalidOutcomeIndex` | Any `idx ≥ outcomeCount` reverts with `InvalidOutcomeIndex` |
| `testFuzz_totalTraderShares_equalsSumOfIndividuals` | Aggregate == sum of individual balances for 3 traders |
| `testFuzz_quantity_equals_epsilon_plus_totalTraderShares` | `q[i] = ε + totalTraderShares[i]` invariant |
| `testFuzz_resolve_payoutMatchesSettleMarketArg` | `settleMarket` arg == `floor(totalTraderShares[winner] / 1e12)` |
| `testFuzz_buy_revert_pastTradingDeadline` | Any warp past deadline causes `TradingClosed` |
| `testFuzz_resolve_revert_pastResolutionDeadline` | Any elapsed ≥ 0 past deadline causes `ResolutionDeadlinePassed` |
| `testFuzz_expire_revert_beforeDeadline` | Any time ≤ `resolutionDeadline` causes `ResolutionDeadlineNotPassed` |

---

### `BlieverMarket_Invariant.t.sol` — Stateful Invariant Tests

**Run:** `forge test --match-contract BlieverMarket_InvariantTest`

**Configure in `foundry.toml`:**
```toml
[invariant]
runs  = 500
depth = 100
fail_on_revert = false
```

#### Handler (`BlieverMarketHandler`)
Wraps all state-mutating market functions. Foundry calls these in random order and length sequences:
- `buy(actorSeed, shareAmt)` — random buy on outcome 0 for one of three traders
- `sell(actorSeed, shareAmt)` — standard sell on outcome 0 (only held shares, no CSS)
- `resolve()` — resolves market once to outcome 0
- `warpTrading(secs)` — advances time within the trading window

**Ghost variables** track cumulative state:
- `ghost_totalBuyShares` — sum of all bought shareAmounts
- `ghost_totalSellShares` — sum of all sold shareAmounts
- `ghost_buyCount`, `ghost_sellCount` — call counters
- `ghost_resolved` — mirrors `resolved` for invariant cross-check

#### Invariants

| ID | Name | Property |
|---|---|---|
| I-1 | `invariant_I1_quantityNonNegative` | `q[i] ≥ 0` for all outcomes (uint256 never wraps) |
| I-2 | `invariant_I2_quantityLedgerConsistency` | `q[0] == ε + totalTraderShares[0]` |
| I-3 | `invariant_I3_totalShares_equalsSumOfIndividuals` | `totalTraderShares[0] == shares[alice][0] + shares[bob][0] + shares[carol][0]` |
| I-4 | `invariant_I4_sumOfPricesGtOne` | `Σ prices > 1e18` while market is active (LS-LMSR spread invariant) |
| I-5 | `invariant_I5_resolvedIsMonotone` | `resolved` can only transition `false → true`, never back |
| I-6 | `invariant_I6_claimedIsMonotone` | `hasClaimed[addr]` can only transition `false → true`, never back |
| I-7 | `invariant_I7_marketStatus_consistent` | `getMarketStatus().tradingOpen` matches `!resolved && timestamp ≤ tradingDeadline` |
| I-8 | `invariant_I8_initialQuantitiesImmutable` | `getInitialQuantities()` always returns original epsilon values |

---

### `BlieverMarket_Gas.t.sol` — Gas Profiling Tests

**Run:** `forge test --match-contract BlieverMarket_GasTest --gas-report`

**Purpose:** Detect gas regressions automatically in CI.

**Measurement strategy:**
- Warm measurements: one prior call warms up all storage slots before the measured call.
- Cold measurements: first call on a fresh market (all slots cold).

**Gas caps (generous upper bounds for CI stability):**

| Function | Scenario | Gas Cap |
|---|---|---|
| `buy()` | 2-outcome warm | 150 000 |
| `buy()` | 2-outcome cold | 250 000 |
| `buy()` | 7-outcome warm | 250 000 |
| `sell()` | standard 2-outcome warm | 150 000 |
| `sell()` | CSS 7-outcome warm | 350 000 |
| `resolve()` | no trades | 80 000 |
| `resolve()` | with one trade | 100 000 |
| `claim()` | winner | 100 000 |
| `getPrice()` | view | 30 000 |
| `getAllPrices()` | 7-outcome view | 80 000 |
| `getBuyCost()` | view | 40 000 |
| `getMarketStatus()` | view | 30 000 |

**CI integration:**
```bash
# Generate baseline
forge snapshot --evm-version cancun

# Fail CI if any gas increased
forge snapshot --check --evm-version cancun

# Show diff against saved snapshot
forge snapshot --diff .gas-snapshot --evm-version cancun
```

---

## What Is NOT Tested Here

These are explicitly out of scope for this test suite (covered elsewhere):

| Excluded Area | Why |
|---|---|
| `BlieverV1Pool` internal logic | Pool has its own test suite (`test/BlieverV1Pool/`) |
| `LSMath` library math correctness | Pure library tested separately (`test/LSMath/`) |
| `BlieverMarketFactory` deploy/clone logic | Factory not yet developed (development phase) |
| Resolution Adapter / UMA oracle integration | Not part of `BlieverMarket.sol` |
| EIP-1967 proxy upgrades | `BlieverMarket` clones are intentionally immutable (no UUPS) |
| EIP-2612 permit happy path | `MockUSDC` does not implement permit; permit tests require a full ERC-20Permit mock |
| Multi-market interactions | Isolation: each test targets one market clone |

---

## Improvement Opportunities

When the codebase matures, consider adding:

1. **Permit fuzz tests** — Deploy a proper EIP-2612 MockUSDC and fuzz the `v/r/s/deadline` permit flow for both `buy()` and CSS-path `sell()`. Include the "permit front-run → fallback to allowance" path.

2. **`InsufficientPermitAllowance` path** — The only uncovered revert in buy/sell. Requires a consumed permit nonce with zero pre-existing allowance.

3. **Fork tests against Base mainnet** — Use a real USDC contract and real pool deployment to verify the end-to-end gas costs on an actual L2 block.

4. **Differential test against Python reference** — Implement the LS-LMSR cost function in Python (or use `ff` library) and compare output against `getBuyCost()` / `getSellEstimate()` for the same input vectors.

5. **Handler expansion for invariant tests** — Add CSS sells, outcome-1 buys, multi-outcome sells, and post-resolution claims to the Handler to increase the reachable state space.

6. **`_NegativeBuyCost` path** — Currently unreachable in normal operation (would indicate an LSMath bug). Consider adding a malicious LSMath mock to explicitly cover the `if (tradeCost18 < 0) revert NegativeBuyCost()` branch.

---

## foundry.toml Reference Configuration

Add to your project's `foundry.toml` for these tests:

```toml
[profile.default]
evm_version = "cancun"   # required: EIP-1153 transient storage

[fuzz]
runs  = 2000
seed  = "0xBELIEVER"

[invariant]
runs            = 500
depth           = 100
fail_on_revert  = false   # handler try-catches handle expected reverts
```
