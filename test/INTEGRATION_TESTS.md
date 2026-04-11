# BlieverMarket × BlieverV1Pool — Integration Test Documentation

> **For:** Test developers, protocol engineers, and auditors who want to understand,
> debug, or extend the integration test suite.  
> **Test folder:** `contracts/test/Integration/`  
> **Stack:** Solidity 0.8.31 · Foundry · Base Chain (Cancun EVM)  
> **Phase:** Development

---

## Overview

These tests verify the **real interaction between `BlieverMarket` and `BlieverV1Pool`** — no mocks.
Every USDC transfer, pool accounting update, and cross-contract state change is exercised
with the production contracts deployed in a fresh Foundry environment.

Unit tests (in `test/BlieverMarket/` and `test/BlieverV1Pool/`) test each contract in isolation
with mock counterparts.  
Integration tests prove that the two contracts work correctly **together** — covering the
full prediction-market lifecycle from LP deposit through to winner payout and LP withdrawal.

---

## Quick Reference

| File | Contract | What it covers |
|---|---|---|
| `IntegrationBase.t.sol` | `IntegrationBase` *(abstract)* | Shared deployment, fixtures, and helpers |
| `Integration_LifeCycle.t.sol` | `Integration_LifeCycle` | Full trade → resolve → claim lifecycle (binary + multi-outcome) |
| `Integration_MultiMarket.t.sol` | `Integration_MultiMarket` | Two concurrent markets, pool accounting, LP withdrawal |
| `Integration_PoolConstraints.t.sol` | `Integration_PoolConstraints` | Capacity, LP limits, pause, deregister, force-settle |

**Run all integration tests:**
```bash
forge test --match-path "test/Integration/**" --evm-version cancun -vvv
```

**Run a specific file:**
```bash
forge test --match-contract Integration_LifeCycle --evm-version cancun -vvv
forge test --match-contract Integration_MultiMarket --evm-version cancun -vvv
forge test --match-contract Integration_PoolConstraints --evm-version cancun -vvv
```

> **EVM version note:** `--evm-version cancun` is mandatory.  
> `BlieverMarket` uses `ReentrancyGuardTransient` which depends on EIP-1153 transient  
> storage (TSTORE / TLOAD), introduced in Cancun.

---

## Shared Parameters

All integration tests inherit these constants from `IntegrationBase`:

| Parameter | Value | Meaning |
|---|---|---|
| `ALPHA` | `3e16` | LS-LMSR commission α = 3 % (18-dec) |
| `MAX_RISK` | `500e6` | Pool risk budget per market = 500 USDC (6-dec) |
| `RESERVE_BPS` | `2000` | LP reserve buffer = 20 % of total assets |
| `EPSILON_2` | `480e18` | Initial seed quantity for 2-outcome market (18-dec) |
| `EPSILON_7` | `355e18` | Initial seed quantity for 7-outcome market (18-dec) |
| `LP_DEPOSIT` | `50_000e6` | USDC deposited by the LP in setUp (50 K USDC) |
| `TRADER_USDC` | `2_000e6` | USDC given to each trader in `_setupTrader` (2 K USDC) |
| `MIN_SHARE` | `1e15` | `BlieverMarket.MIN_SHARE_AMOUNT` (dust guard) |
| `SHARE_TO_USDC` | `1e12` | Conversion: 1e18 shares = 1e6 USDC at settlement |

**Time parameters** (set relative to Foundry's default `block.timestamp`):

| Variable | Value |
|---|---|
| `tradingDeadline` | `block.timestamp + 7 days` |
| `resolutionDeadline` | `block.timestamp + 14 days` |

---

## `IntegrationBase.t.sol` — Shared Infrastructure

**Role:** Abstract base contract inherited by all three integration test files.

### What is deployed in `setUp()`

1. **`IntegrationUSDC`** — 6-decimal ERC-20 with a permissionless `mint()`. Mirrors the USDC
   interface used on Base mainnet.

2. **`BlieverV1Pool`** — UUPS proxy (via `ERC1967Proxy`) backed by a freshly deployed
   implementation. Initialized with `ALPHA`, `MAX_RISK`, `RESERVE_BPS`, and `admin` as the
   privileged address.

3. **`BlieverMarket` master implementation** — EIP-1167 `_disableInitializers()` prevents
   use of the master directly; all live markets are minimal-proxy clones.

4. **LP seed deposit** — `LP_DEPOSIT` (50 K USDC) minted to `lp` and deposited into the
   pool, giving the vault enough capital to register multiple markets.

### Helper Functions

| Helper | Purpose |
|---|---|
| `_lpDeposit(lp, amount)` | Mints USDC, approves pool, calls `pool.deposit()`. |
| `_deployMarket(nOutcomes, epsilon)` | Clones the market impl, calls `initialize()`, then calls `pool.registerMarket()` as `admin`. The test contract (`address(this)`) is set as the market's `factory` so tests can call `expireUnresolved()`, `pause()`, and `unpause()` directly. |
| `_setupTrader(trader)` | Mints `TRADER_USDC` to trader and grants max USDC approval to the **pool** (not the market — `collectTradeCost` pulls from the pool's address). |
| `_buy(market, trader, outcomeIndex, shareAmount)` | Gets cost estimate from `getBuyCost()`, mints more USDC if needed, calls `market.buy(…, type(uint256).max, …)`. Returns actual USDC deducted. |
| `_sell(market, trader, outcomeIndex, shareAmount)` | Gets CSS cost estimate from `getSellEstimate()`, mints USDC if a net cost applies, calls `market.sell(…, 0, type(uint256).max, …)`. |
| `_resolve(market, winningOutcome)` | Pranks `resolver` and calls `market.resolve()`. |
| `_claim(market, winner)` | Pranks `winner` and calls `market.claim()`. |
| `_expireUnresolved(market)` | Calls `market.expireUnresolved()` from `address(this)` (factory role). |
| `_assertPoolSolvent()` | Asserts `pool.isSolvent() == true` AND `USDC.balanceOf(pool) ≥ pool.totalLiability()`. |

### Approval Flow

Traders approve the **pool address** (not the market), because:

```
market.buy() → pool.collectTradeCost() → IERC20(asset).safeTransferFrom(trader, pool, cost)
```

`_setupTrader` sets `usdc.approve(address(pool), type(uint256).max)` for each trader.

---

## `Integration_LifeCycle.t.sol` — Full Lifecycle Tests

Tests the end-to-end prediction-market lifecycle across both a binary (2-outcome) and a
multi-outcome (7-outcome) market.

### Test Matrix

#### Binary Market (2 outcomes)

| Test | What it verifies |
|---|---|
| `test_binary_fullLifeCycle_winnerBuysAndClaims` | Trader buys winning outcome → USDC flows from trader to pool on buy → pool correctly records settled payout → winner claims USDC back → MARKET_ROLE auto-revoked. |
| `test_binary_loserBuysWrongOutcome_cannotClaim` | Trader holds shares of the losing outcome; `claim()` reverts with `NoWinningShares`; trader's USDC never returns (absorbed as LP profit). |
| `test_binary_partialSell_remainingSharesWin` | Buy 10e18, sell 5e18 (refund received from pool), then claim the remaining 5e18 position. Confirms sell refund lowers pool balance and remaining shares still entitle claim. |
| `test_binary_noTrades_settlesWithZeroPayout` | Market resolved with zero `_totalTraderShares` → `settledPayout = 0` → full `riskBudget` released → MARKET_ROLE revoked immediately (zero-payout path in `settleMarket`). |
| `test_binary_multipleTraders_onlyWinnerClaims` | Three traders buy; two on the winning side claim; the loser cannot claim. |
| `test_binary_exactUSDCFlows_endToEnd` | Verifies exact USDC arithmetic: `pool net = initialBalance + cost − payout`; `alice net = initialBalance − cost + payout`. |

#### 7-Outcome Market

| Test | What it verifies |
|---|---|
| `test_7outcome_multipleBuyers_correctWinnerClaims` | Three traders on three different outcomes; correct winner resolved; two non-winners get `NoWinningShares`. |
| `test_7outcome_CSSSell_translatesPosition_winnerClaims` | Alice holds 5e18 of outcome 0 and sells 10e18 → CSS fires with `tBar = 5e18` → Alice's outcome-0 balance becomes 0 and she receives 5e18 on each of outcomes 1–6. Resolving outcome 1 lets her claim `floor(5e18 / 1e12) = 5 USDC`. |
| `test_expireUnresolved_zeroPayout_traderCannotClaim` | Factory calls `expireUnresolved()` after `resolutionDeadline + 1`. Pool settles with 0 payout. Trader who bought outcome 0 cannot claim because `winningOutcome = outcomeCount` (expiry sentinel). |

#### Liability Tracking

| Test | What it verifies |
|---|---|
| `test_binary_liabilityBounded_afterMultipleBuys` | `currentLiability ≤ riskBudget` always holds after many buys, confirming the pool's belt-and-suspenders cap in `collectTradeCost`. |
| `test_binary_buyAllSellAll_poolRemainsSolvent` | Buy 10e18 then sell all 10e18 back. After a full round-trip, trader's share balance is zero and pool is solvent. |

---

## `Integration_MultiMarket.t.sol` — Concurrent Markets

Tests the pool with two markets active simultaneously: `marketA` (2-outcome) and
`marketB` (7-outcome).

### Test Matrix

| Test | What it verifies |
|---|---|
| `test_twoMarkets_poolAccountingOnRegistration` | `activeMarketCount == 2`, `totalLiability == 2 × MAX_RISK` immediately after both markets register. |
| `test_twoMarkets_tradeOnA_doesNotAffectBMarketInfo` | Trades on market A leave market B's `currentLiability`, `riskBudget`, and `hasTrades` fields unchanged — per-market accounting is fully isolated. |
| `test_twoMarkets_totalLiabilityEqualsSum` | After trading on both, `pool.totalLiability() == marketA.currentLiability + marketB.currentLiability`. |
| `test_twoMarkets_settleA_keepsBActive` | Settling market A decrements `activeMarketCount` to 1 and reduces `totalLiability` by A's live liability at settlement time. Market B remains fully tradeable. |
| `test_twoMarkets_sequentialFullLifecycle_poolCleanAfter` | Full lifecycle of both markets sequentially: after both settle and claims are processed, `activeMarketCount == 0` and `totalLiability == 0`. |
| `test_twoMarkets_solvencyInvariant_throughoutConcurrentTrading` | `_assertPoolSolvent()` is called at 10 checkpoints throughout a sequence of interleaved buys, sells, a settle, and claims across both markets. |
| `test_twoMarkets_lpWithdrawal_increasesAsMarketsSettle` | LP's `maxWithdraw` grows stepwise as market A settles, then again as market B settles, confirming riskBudget is correctly released each time. |
| `test_twoMarkets_lpWithdrawal_cappedWhileMarketsActive` | `pool.maxWithdraw(lp) ≤ pool.availableLiquidity()` and `< LP_DEPOSIT` while both markets are active; a withdrawal of exactly `maxWd` succeeds. |
| `test_twoMarkets_navIncreases_afterProfitableSettlement` | Both traders lose (buy wrong outcome); vault absorbs their cost as LP profit. `pool.nav()` after settlement > `pool.nav()` before. |

---

## `Integration_PoolConstraints.t.sol` — Pool Constraints & Administration

Tests edge cases at the pool boundary: capacity limits, LP withdrawal maths, pause mechanics,
deregistration, and the emergency force-settle path.

### Test Matrix

#### Capacity / Registration

| Test | What it verifies |
|---|---|
| `test_registerMarket_reverts_CapacityExceeded_insufficientAssets` | A pool with only 624 USDC (activeCap = 499.2e6 < MAX_RISK = 500e6) reverts with `CapacityExceeded` on `registerMarket`. |
| `test_registerMarket_succeedsAfterSufficientDeposit` | Adding ≥ 625 USDC unlocks the capacity threshold; `registerMarket` succeeds. |

#### LP Withdrawal Constraints

| Test | What it verifies |
|---|---|
| `test_lpWithdrawal_cappedByActiveMarketLiability` | `maxWithdraw ≤ availableLiquidity < LP_DEPOSIT` while a market is active; free liquidity increases after market settles. |
| `test_lp_partialWithdrawal_succeedsWhileMarketActive` | Withdrawing exactly `maxWithdraw` succeeds while a market is live; pool remains solvent. |
| `test_lp_maxWithdraw_isMaxAfterAllMarketsSettle` | After all markets settle, LP can withdraw the full `maxWithdraw` amount. |
| `test_lpFreeLiquidity_mayIncreaseAsVolumeGrows` | After balanced buys on both outcomes, `currentLiability ≤ MAX_RISK` (cap invariant holds). |

#### Pause Mechanics

| Test | What it verifies |
|---|---|
| `test_pause_blocksTrades_doesNotBlockSettlementOrClaim` | Pool pause blocks `market.buy()` (which calls `pool.collectTradeCost`). `market.resolve()` and `market.claim()` succeed unpaused — neither `settleMarket` nor `claimWinnings` are pause-gated. |
| `test_pause_roleEnforcement` | Non-PAUSER_ROLE cannot pause; non-DEFAULT_ADMIN_ROLE cannot unpause. |

#### Deregister Market

| Test | What it verifies |
|---|---|
| `test_deregisterMarket_noTrades_restoresLiabilityAndCount` | `deregisterMarket` on a trade-free market reduces `totalLiability` by `MAX_RISK`, decrements `activeMarketCount`, revokes `MARKET_ROLE`, and deletes the `MarketInfo` entry. |
| `test_deregisterMarket_reverts_MarketHasTrades` | Attempting to deregister a market that has received at least one trade reverts with `MarketHasTrades`. |

#### Force Settle (Emergency)

| Test | What it verifies |
|---|---|
| `test_forceSettle_releasesLiability_revokesMARKET_ROLE` | `forceSettleMarket` zeros `settledPayout`, `riskBudget`, and `currentLiability`; releases the market's live liability from `totalLiability`; revokes `MARKET_ROLE`. |
| `test_forceSettle_marketClaimReverts_MarketNotResolved` | After force-settle, `market.resolved == false` (pool-only operation, never calls into market contract). `market.claim()` reverts with `MarketNotResolved`. |
| `test_forceSettle_revertsForNonEmergencyRole` | Non-EMERGENCY_ROLE callers cannot force-settle. |

#### Solvency on Refund

| Test | What it verifies |
|---|---|
| `test_sell_refund_poolRemainssolvent` | After a large sell refund (pool transfers USDC out via `distributeRefund`), the pool balance is still ≥ `totalLiability` — confirming `_assertSolvent()` inside `distributeRefund` passes. |

---

## Key Cross-Contract Flows Tested

```
BUY FLOW
  market.buy() ─→ pool.collectTradeCost(trader, cost, newLiability)
                      ├── IERC20.safeTransferFrom(trader, pool, cost)  [USDC movement]
                      └── totalLiability updated by delta               [accounting]

SELL REFUND FLOW
  market.sell() ─→ pool.distributeRefund(trader, refund, newLiability)
                      ├── IERC20.safeTransfer(pool → trader, refund)   [USDC movement]
                      ├── totalLiability updated by delta               [accounting]
                      └── _assertSolvent()                              [invariant check]

SETTLEMENT FLOW
  market.resolve() ─→ pool.settleMarket(totalPayout)
                          ├── info.settled = true                        [state]
                          ├── totalLiability -= currentLiability         [accounting]
                          ├── activeMarketCount--                        [accounting]
                          └── MARKET_ROLE revoked if payout == 0         [access control]

CLAIM FLOW
  market.claim() ─→ pool.claimWinnings(winner, amount)
                        ├── IERC20.safeTransfer(pool → winner, amount)  [USDC movement]
                        ├── info.claimedPayout += amount                 [accounting]
                        └── MARKET_ROLE revoked if fully claimed         [access control]
```

---

## What Is NOT Covered Here

These are explicitly out of scope for this integration suite:

| Excluded Area | Where it is tested |
|---|---|
| Individual function reverts (invalid inputs, role guards) | `test/BlieverMarket/` and `test/BlieverV1Pool/` unit tests |
| LS-LMSR math precision (cost function, price formula) | `test/LSMath/` unit tests |
| Fuzz / invariant coverage for the market in isolation | `BlieverMarket_Fuzz.t.sol`, `BlieverMarket_Invariant.t.sol` |
| Invariant fuzzing of pool accounting | `BlieverV1Pool_Invariants.t.sol` |
| `BlieverMarketFactory` deploy / clone logic | Factory not yet developed |
| EIP-2612 permit happy-path and front-run revert | `BlieverMarket_Trading.t.sol` |
| ERC-4626 share accounting precision and rounding | `BlieverV1Pool` unit tests |
| UUPS upgrade storage collision / state preservation | Upgrade tests (not yet written) |

---

## `foundry.toml` Configuration

```toml
[profile.default]
evm_version = "cancun"   # required: EIP-1153 transient storage

[fuzz]
runs = 2000
seed = "0xBELIEVER"
```

---

## Extending the Suite

### Adding a New Lifecycle Test

1. Inherit `IntegrationBase` in a new `Integration_<Area>.t.sol` file.
2. Deploy markets via `_deployMarket(nOutcomes, epsilon)` in `setUp()`.
3. Set up traders with `_setupTrader(addr)`.
4. Use `_buy / _sell / _resolve / _claim / _expireUnresolved` helpers for state transitions.
5. Call `_assertPoolSolvent()` at every meaningful checkpoint.

### Adding a New Market Scenario

- For a new outcome count `n`, pre-compute `ε = R / (1 + α·n·ln n)` in 18-dec and add
  it as a constant (e.g. `EPSILON_5` for a 5-outcome market).
- Pass the new `(n, ε)` pair to `_deployMarket()`.

### Extending Market Count in Multi-Market Tests

- The current suite uses at most 2 concurrent markets (500e6 × 2 = 1000e6 liability).
  With `LP_DEPOSIT = 50_000e6` and `RESERVE_BPS = 2000`, the pool can support up to
  `floor(50_000e6 × 0.8 / 500e6) = 80` concurrent markets.
- Raise the number of `_deployMarket()` calls and verify `pool.activeMarketCount()` accordingly.
