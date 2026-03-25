# BlieverMarket — Architecture & System Design

> **File:** `contracts/src/BlieverMarket.sol`  
> **Interface:** `contracts/src/interfaces/IBlieverV1Pool.sol`  
> **Scope:** Market Logic Layer — Pure AMM wrapper, Internal Ledger, Trade Routing, Settlement Trigger.

---

## 1. Overview & Purpose

`BlieverMarket` is the **ephemeral, lightweight master implementation contract** that represents a single prediction market event. It is designed to be deployed thousands of times via the **EIP-1167 Minimal Proxy standard** using `CREATE2` deterministic addressing by the `BlieverMarketFactory`.

Its sole responsibility is to be the **mathematical brain** of a single market: tracking the AMM state, enforcing the Covered Short Selling rule on individual traders, computing trade costs via `LSMath`, and routing USDC movements to the central `BlieverV1Pool` vault. It holds **no USDC** itself.

> **Core Principle: Pure Math Wrapper.** Every real dollar lives in `BlieverV1Pool`. `BlieverMarket` is a stateful coordination layer between the trader, the LS-LMSR mathematics, and the vault.

---

## 2. The Protocol Stack — Where BlieverMarket Sits

```
┌──────────────────────────────────────────────────────────────────────┐
│  TRADER / FRONTEND                                                   │
│  (calls buy/sell/claim; approves BlieverV1Pool for USDC)            │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────────┐
│  BlieverMarket (Clone N)         ← THIS CONTRACT                    │
│  • Holds the q-vector (AMM state)                                   │
│  • Holds the internal ledger (per-trader shares)                    │
│  • Computes costs via LSMath (pure, stateless library)              │
│  • Routes USDC to/from BlieverV1Pool                                │
└──────┬──────────────────────────────────────┬────────────────────────┘
       │ MARKET_ROLE calls                    │ calls from
       │ collectTradeCost / distributeRefund  │ resolve()
       │ settleMarket / claimWinnings         │
       ▼                                      ▼
┌──────────────────┐                 ┌─────────────────────────────────┐
│  BlieverV1Pool   │                 │  Resolution Adapter             │
│  (ERC-4626 vault)│                 │  (UMA OO / Reality.eth / etc.)  │
│  All USDC lives  │                 │  Only entity that calls resolve()│
│  here            │                 └─────────────────────────────────┘
└──────────────────┘
       │ LSMath (library, inlined at compile-time)
       └──────────────────────────────────────────
         • liquidityParameter(q, α)
         • costFunction(q, α)
         • calculateTradeCostDetailed(qFrom, qTo, α)  → (tradeCost, costTo)
         • calculateTradeCost(qFrom, qTo, α)          → tradeCost  [view path only]
         • calculateWorstCaseLossFromCosts(costCurrent, costInitial, q)
         • calculateWorstCaseLoss(q, q⁰, α)           [retained for completeness]
         • getPrice(q, i, α)  /  getAllPrices(q, α)
```

---

## 3. LS-LMSR Mathematical Foundation

### 3.1 The Quantity Vector (q-vector)

The market's internal state is captured by a single vector **q** ∈ ℝⁿ₊, where:

- **n** = number of mutually exclusive outcomes
- **qᵢ** = total shares outstanding for outcome i across all traders
- All components are **non-negative** (positive orthant invariant)

The vector starts at **q⁰ = [ε, ε, …, ε]** (the initial seed), and evolves with every trade.

### 3.2 The Cost Function C(q)

The LS-LMSR cost function is:

$$C(\mathbf{q}) = b(\mathbf{q}) \cdot \ln\!\left(\sum_{i=1}^{n} \exp\!\left(\frac{q_i}{b(\mathbf{q})}\right)\right)$$

where the **liquidity parameter** is:

$$b(\mathbf{q}) = \alpha \cdot \sum_{i=1}^{n} q_i$$

This is the fundamental departure from standard LMSR: **b is variable**, growing with total market volume. As volume grows, the market becomes less elastic (prices less sensitive to individual trades), and the AMM's worst-case loss bound shrinks in real-time.

### 3.3 The Initial Seed ε and Risk Budget R

The Market Factory computes ε off-chain such that **C(q⁰) = R**, where R is the vault's `maxRiskPerMarket`:

$$\varepsilon = \frac{R}{1 + \alpha \cdot n \cdot \ln n}$$

This satisfies the LS-LMSR initialisation condition (Proposition 4.9): from block zero, the market maker's worst-case loss is exactly R, the amount the vault reserved at market registration.

**In practice** (2-outcome market, α = 3%, R = $10,000):
$$\varepsilon = \frac{10{,}000}{1 + 0.03 \cdot 2 \cdot \ln 2} \approx \$9{,}601$$

This means approximately $9,601 of initial "seed" capital is embedded in each outcome as the vault's market-making commitment. This is **NOT trader capital** — it is the vault's LS-LMSR cost of providing initial liquidity.

**`_initialCost` — The Cached C(q⁰):**  Because q⁰ is immutable after `initialize()`, the value `C(q⁰)` is a per-market constant. `initialize()` evaluates it once and writes it to `_initialCost` (a single storage slot). Every subsequent buy and sell reads `_initialCost` via one warm SLOAD rather than reloading the n-element `_initialQuantities` array and rerunning the full `exp`/`ln` loop for `costFunction(q⁰)`. On a 10-outcome market this is approximately **40,000–60,000 gas saved per trade** on the worst-case-loss step alone.

### 3.4 Trade Pricing

The cost of a trade moving the market from **q_old** to **q_new** is:

$$\text{cost} = C(\mathbf{q}_{\text{new}}) - C(\mathbf{q}_{\text{old}})$$

- **Positive cost** → trader pays the vault (buy trade).
- **Negative cost** → vault refunds the trader (sell trade).

This is path-independent (a key LS-LMSR property): the cost depends only on the start and end states, never on the order of intermediate trades.

In the hot trading path, this is computed via `LSMath.calculateTradeCostDetailed(qOld, qNew, α)`, which returns both the cost difference **and** `C(qNew)` in a single pass. The returned `C(qNew)` is immediately reused for the vault liability update, ensuring `costFunction(qNew)` is evaluated exactly once per trade.

---

## 4. Covered Short Selling (CSS) — The Internal Ledger

### 4.1 Why CSS Is Necessary

A fundamental vulnerability of path-independent AMMs is the potential for riskless arbitrage: if buying one share of every outcome costs more than $1, a trader could buy all outcomes and immediately sell the guaranteed $1 payout back to the AMM for risk-free profit.

LS-LMSR prices sum to **> 1** (this is the AMM's spread/revenue). To prevent the arbitrage, the market must enforce that the obligation vector always moves **forward** — all components remain non-negative.

### 4.2 The CSS Scheme (Othman et al., Section 3.3.2)

Rather than the punitive "No Selling" scheme (which forces traders to buy ALL complementary outcomes to exit a position), the protocol uses **Covered Short Selling**:

> Traders may sell shares they already own. Naked short-selling against the AMM is blocked. If a sell would make the trader's per-outcome balance negative, an automatic translation is applied.

**Formal Definition:**

Let **q^t** = the trader's current per-outcome balance vector.
Trader requests to sell `shareAmount` of outcome `i`:

```
desired_delta[i] = −shareAmount
desired_delta[j≠i] = 0
```

If `q^t[i] < shareAmount` (trader would go negative):

```
t_bar = shareAmount − q^t[i]   // CSS translation scalar
actual_delta[i]   = −shareAmount + t_bar = −q^t[i]   // sell everything held
actual_delta[j≠i] = +t_bar                            // buy t_bar of all others
```

The trader's resulting position:
- `q^t[i]`  → 0 (sold all they owned)
- `q^t[j≠i]` → `q^t[j] + t_bar` (acquired guaranteed coverage)

All components remain ≥ 0. ✓

### 4.3 The Internal Ledger Rule

CSS requires that the smart contract knows **every trader's exact per-outcome balance** at all times. This mandates the **Internal Ledger Rule**:

> Outcome shares are stored as non-transferable internal `mapping(address ⇒ mapping(uint256 ⇒ uint256))` balances. No ERC-20 `transfer()`, `approve()`, or `transferFrom()` methods exist.

**Why no ERC-20?** If Trader A transfers shares to Trader B off-platform (peer-to-peer ERC-20 transfer), the contract's ledger for Trader B shows zero, but Trader B holds real shares. When Trader B tries to sell, the contract interprets it as naked short-selling and misapplies the CSS translation, breaking the AMM's mathematical invariant.

**Regulatory alignment (2026):** The Internal Ledger ensures all positions are KYC-bound. Only addresses that passed the platform's compliance gateway can acquire, hold, or redeem market positions. This satisfies AML/TDS reporting requirements without secondary market leakage.

### 4.4 CSS Trade Accounting Invariant

The following invariant is maintained after every trade:

$$\sum_{\text{all traders}} q^t_i = q_i - \varepsilon_i$$

Where εᵢ is the initial epsilon seed for outcome i. The market's q-vector includes εᵢ (used purely for AMM math), but `_totalTraderShares[i]` tracks only actual trader positions. This separation is **critical** for correct payout calculation at settlement.

---

## 5. Resolution Architecture — The Adapter Interface

`BlieverMarket` is deliberately **oracle-agnostic**. It accepts resolution signals only from one pre-configured address: the `resolver` (the Resolution Adapter).

```
resolve(uint8 winningOutcome)   ←── called by Resolution Adapter
```

This separation achieves two goals:

1. **Upgradeable oracle sources:** By 2026, the protocol may migrate from UMA OOv2 → UMA OOv3 → Reality.eth or any other oracle without touching the market contracts. Only the factory needs to deploy clones pointing to a new resolver address.

2. **Minimal attack surface:** The market holds no oracle formatting logic, no `ancillaryData` parsing, no DVM escalation paths. These concerns live entirely in the Resolution Adapter, which is independently auditable.

### 5.1 Resolution Settlement Flow

```
Oracle finalizes outcome
      │
      ▼
Resolution Adapter calls:
  BlieverMarket.resolve(winningOutcome)
      │
      ├── Mark: resolved = true, winningOutcome = W
      │
      ├── Compute: totalPayoutUsdc = _totalTraderShares[W] / SHARE_TO_USDC
      │            (excludes epsilon seed — only real trader positions)
      │
      └── Call: BlieverV1Pool.settleMarket(totalPayoutUsdc)
                  ├── Sets: info.settled = true
                  ├── Sets: info.settledPayout = totalPayoutUsdc
                  ├── Zeroes: info.currentLiability → totalLiability decreases → LP NAV rises
                  └── Emits: MarketSettled(market, totalPayout, profit)
```

### 5.2 Emergency Expiry Path

When `resolutionDeadline` passes without oracle resolution, the factory calls `expireUnresolved()`:

```
factory calls: BlieverMarket.expireUnresolved()
      │
      ├── Mark: resolved = true  (winningOutcome stays 0 — no legitimate claim possible)
      │
      ├── Call: BlieverV1Pool.settleMarket(0)
      │           └── settledPayout = 0; MARKET_ROLE immediately revoked; vault absorbs riskBudget
      │
      └── Emit: MarketExpired(factory, timestamp, ExpiryReason.TIMEOUT)
               └── TIMEOUT = resolutionDeadline passed with no oracle resolution
                   ExpiryReason enum is reserved for future oracle-error variants
                   without ABI breakage. Off-chain indexers should handle unknown values.
```

### 5.3 The Epsilon Exclusion Design

Why exclude epsilon from `totalPayoutUsdc`?

The epsilon shares in the q-vector are the vault's own market-making seed — they represent the vault's obligation to itself, not to any trader. If epsilon were included in `settledPayout`, then after all legitimate traders claim, `claimedPayout < settledPayout` by the epsilon amount, and the pool would **never auto-revoke** `MARKET_ROLE` from the market contract.

By computing `totalPayoutUsdc = _totalTraderShares[winningOutcome] / SHARE_TO_USDC`, the `settledPayout` exactly equals the sum of all legitimate winner payouts. When the last winner claims, `claimedPayout == settledPayout` and the pool auto-revokes `MARKET_ROLE`. The epsilon cost is silently absorbed into LP NAV as the vault's market-making operating cost — consistent with the LS-LMSR loss bound theorem.

---

## 6. EIP-1167 Clone Architecture

### 6.1 Why Minimal Proxy?

Prediction markets are **ephemeral**: one event, one contract, one resolution. Deploying full 24 KB contracts for every political election or sports outcome at 500,000+ gas per deployment is economically non-viable at scale.

EIP-1167 solves this with a **45-byte shell** that forwards all calls via `DELEGATECALL` to the master `BlieverMarket` implementation:

```
Cost comparison (approximate, Base chain):
  Full deployment:         ~400,000 gas  (~$0.08 at typical Base fees)
  EIP-1167 clone:           ~41,000 gas  (~$0.008 at typical Base fees)
  Gas savings per market:       ~90%
```

All **logic** lives in the master. All **state** lives in each clone's own storage context. This guarantees mathematical consistency across all markets while enabling mass deployment.

### 6.2 CREATE2 Deterministic Addressing

The factory uses `CREATE2` so that market addresses are **pre-computable** before deployment. This enables:

- Off-chain order books to reference markets before they are on-chain
- Lazy deployment (deploy only when the first trade occurs)
- Frontend interfaces to pre-register trading pairs
- Counterfactual market analysis without on-chain gas costs

### 6.3 Why No UUPS Upgrade in Market Contracts?

Market contracts are **intentionally not upgradeable**. Traders must have cryptographic guarantees that the mathematical rules and resolution parameters of a specific market are immutable from the moment of their first trade. A compromised admin key must never be able to alter the cost function, change the resolver, or modify the outcome set mid-market.

> *Security guarantee*: The market's rules are set in stone at `initialize()`. The only authorised state transitions are trades, resolution, and claim — all governed by the mathematical constraints of LS-LMSR.

---

## 7. USDC Flow Diagram

```
                     BUY TRADE
                     ─────────
  Trader ──USDC──▶  BlieverV1Pool
    │   (safeTransferFrom, approved by trader — via direct approve OR EIP-2612 permit)
    │
    └── BlieverMarket (no USDC touched)
          │  1. Optional: IERC20Permit(usdc).permit(trader, pool, amount, deadline, v,r,s)
          │     ├─ `usdc` read from cached storage slot — no external call to pool
          │     └─ Falls back silently to existing allowance if nonce consumed
          │  2. collectTradeCost(trader, cost, newLiability)
          └──────────────────────────────────────────────▶ Pool executes safeTransferFrom

                     SELL TRADE (refund)
                     ──────────────────
  Trader ◀──USDC──  BlieverV1Pool
    │   (safeTransfer to trader)
    │
    └── BlieverMarket (no USDC touched)
          │  distributeRefund(trader, refund, newLiability)
          └──────────────────────────────────────────────▶ Pool executes transfer

                     SELL TRADE (CSS net cost)
                     ─────────────────────────
  Trader ──USDC──▶  BlieverV1Pool
    │   (safeTransferFrom, approved by trader — via direct approve OR EIP-2612 permit)
    │
    └── BlieverMarket (no USDC touched)
          │  1. Optional: IERC20Permit(usdc).permit(trader, pool, maxCostUsdc, deadline, v,r,s)
          │     ├─ Attempted ONLY when isRefund == false (net-cost CSS path)
          │     ├─ `usdc` read from cached storage slot — no external call to pool
          │     └─ Falls back silently to existing allowance if nonce consumed
          │  2. collectTradeCost(trader, cost, newLiability)
          └──────────────────────────────────────────────▶ Pool executes safeTransferFrom

                     CLAIM WINNINGS
                     ─────────────
  Winner ◀──USDC──  BlieverV1Pool
    │   (safeTransfer to winner)
    │
    └── BlieverMarket (no USDC touched)
          │  claimWinnings(winner, amount)
          └──────────────────────────────────────────────▶ Pool executes transfer
```

---

## 8. Security Properties & Guarantees

| Property | Mechanism |
|---|---|
| **Solvency** | Pool's `riskBudget = C(q⁰) ≥ C(q) − max(qᵢ)` always (Prop. 4.9). Market never requests more than `riskBudget`. |
| **No naked shorts** | CSS enforces q^t ≥ 0 for all traders at all times. Internal ledger prevents bypass. |
| **Oracle independence** | `resolver` is the only address that may call `resolve()`. Resolution adapter is separately auditable. |
| **Claim once** | `_claimed[address]` guard prevents double-claim. Written before `claimWinnings` call (CEI). |
| **Reentrancy** | `ReentrancyGuardTransient` (EIP-1153) on all state-mutating external functions. Transient storage resets each transaction; no persistent slot consumed. |
| **Immutable rules** | No UUPS upgrade. Factory can only pause/expire — cannot change math or resolver. |
| **Pause asymmetry** | Trading pauses do NOT block `resolve()` or `claim()`. Markets always settle. |
| **Vault protection** | Buy costs round UP (ceil). Refunds round DOWN (floor). Liability uses floor. All vault-protective. `sell()` enforces slippage on both directions: `minRefundUsdc` guards the refund side; `maxCostUsdc` guards the CSS cost side. |
| **Dust prevention** | `buy()` and `sell()` enforce `shareAmount ≥ MIN_SHARE_AMOUNT` (1e15). Prevents state pollution and event spam from sub-economical positions on cheap L2 gas. |
| **Dust claim resolution** | Winners whose shares floor to 0 USDC (< 1 USDC-wei) are not silently locked out. `claim()` marks their position as processed (`_claimed = true`) and emits `DustForfeited`. No USDC is transferred and no revert occurs. The dust value is already excluded from `settledPayout` via identical floor division in `resolve()` — it is absorbed into LP NAV as part of the market-making cost. |
| **Permit griefing resistance** | `buy()` and `sell()` (CSS cost path) both accept an optional EIP-2612 permit. If the signature is consumed by a front-runner, the call falls back to any pre-existing allowance rather than reverting. A trade is never bricked by a consumed nonce. On standard refund sells the permit code is never reached, saving gas on the common path. |

---

## 9. Lifecycle State Machine

```
                        ┌─────────────────────────┐
                        │  INITIALISED             │
                        │  q = q⁰ (epsilon seed)  │
                        │  resolved = false        │
                        └────────────┬────────────┘
                                     │ First trade arrives
                                     ▼
                        ┌─────────────────────────┐
                        │  ACTIVE TRADING          │
                        │  q evolves with trades  │
                        │  deadline = tradingDeadline│
                        └────────────┬────────────┘
                          ┌──────────┴──────────┐
                          │                     │
              block.timestamp >          resolver calls
              tradingDeadline            resolve(winner)
                          │                     │
                          ▼                     ▼
              ┌─────────────────┐   ┌──────────────────────┐
              │ PENDING RESOLVE │   │     RESOLVED         │
              │ (oracle window) │   │  settled=true in pool│
              └────────┬────────┘   │  winners may claim   │
                       │            └──────────┬───────────┘
           resolutionDeadline passes            │
           without oracle resolution    All winners claim
                       │                       │
                       ▼                       ▼
              ┌─────────────────┐   ┌──────────────────────┐
              │ factory calls   │   │  FULLY SETTLED        │
              │ expireUnresolved│   │  MARKET_ROLE revoked  │
              │ settleMarket(0) │   │  from pool           │
              └─────────────────┘   └──────────────────────┘
```

---

## 10. Gas Architecture Notes

- **Single-slot buy write:** `buy()` writes only `_quantities[outcomeIndex]` — the single slot that changed. The full `_storeQuantities(qNew, n)` loop (n SSTOREs) is not used on the buy path. For a 10-outcome market this removes 9 unnecessary warm SSTOREs (~900 gas) per buy.
- **Combined load-and-build for buys:** `_loadQuantitiesForBuy(n, idx, delta)` reads all n quantity slots and simultaneously populates both `qOld` and `qNew` in one loop, replacing the former two-pass pattern (`_loadQuantities` then `_copyArray`). Memory traversal is halved on the buy path.
- **Combined load-and-build for sells:** `_loadQuantitiesForSell(n, idx, netReduce, tBar)` mirrors the buy optimisation for the sell path. It reads all n quantity slots once and simultaneously constructs `qOld` and `qNew` with the CSS mutation applied inline, replacing the former three-step pattern (`_loadQuantities` + `_copyArray` + CSS loop). The `InsufficientMarketQuantity` guard is embedded at the single affected slot, eliminating a redundant branch.
- **Single-slot trader balance read on sell:** `sell()` reads only `_shares[trader][outcomeIndex]` (one mapping SLOAD) to compute `tBar` and update the sold-outcome position. The full `_loadTraderShares(trader, n)` load (n mapping SLOADs) is no longer used in the write path — for a 10-outcome market this removes 9 cold mapping SLOADs (~18,900 gas) on every standard sell.
- **`costFunction(qNew)` evaluated once per trade:** `buy()` and `sell()` both call `LSMath.calculateTradeCostDetailed(qOld, qNew, α)`, which returns `(tradeCost18, costNew)` — the cost difference **and** `C(qNew)` — in a single pass. The `costNew` value is passed directly to `calculateWorstCaseLossFromCosts`, so `costFunction(qNew)` runs exactly once. The previous pattern called `calculateTradeCost` (which computed `costFunction(qNew)` internally) and then `calculateWorstCaseLoss` (which recomputed `costFunction(qNew)` a second time). On a 10-outcome market this eliminates one full `exp`/`ln` loop — approximately **40,000–60,000 gas saved per trade**.
- **`C(q⁰)` cached at initialization:** `initialize()` computes `_initialCost = LSMath.costFunction(initQ, α)` once and stores it in a single storage slot. Every buy and sell reads `_initialCost` via a single warm SLOAD (100 gas) instead of calling `_loadInitialQuantities(n)` (n cold SLOADs ≈ 2,100 gas each) followed by a full `costFunction(q⁰)` evaluation. `q⁰` never changes after `initialize()`, making this computation a constant that belongs in storage, not in the trading hot path.
- **`calculateWorstCaseLossFromCosts` skips redundant math:** With both `costNew` and `_initialCost` already available, the liability helper only needs an O(n) scan of `qNew` to find `max(qᵢ)`. No `exp` or `ln` is executed at all during the liability step.
- **Memory caching for sells:** The q-vector is loaded from storage into memory once at the top of `sell()` via `_loadQuantitiesForSell`, which simultaneously builds the CSS-translated `qNew`. The mutated vector is written back in a single `_storeQuantities` loop. The full write-back is necessary on the sell path because CSS may change multiple slots.
- **Sell permit gated on cost path:** The EIP-2612 permit call in `sell()` is wrapped inside `if (!isRefund)`, so the `usdc` slot read and external `permit()` call are completely bypassed for all standard refund sells. Gas overhead for the permit infrastructure is zero on the common path.
- **Packed storage slots:** The most frequently accessed variables (`pool`, `outcomeCount`, `resolved`, `winningOutcome`) share a single 32-byte slot, yielding a single SLOAD per trade for all four reads.
- **`unchecked` increments:** Loop counters in `buy()`, `sell()`, and all internal helpers use `unchecked { ++i; }` since overflow at uint256 range is impossible for `outcomeCount ≤ 100`. The sold-outcome balance arithmetic in `sell()` also uses `unchecked` where underflow is proven impossible by the CSS invariant.
- **No ERC-20 balance queries:** All USDC transfers are delegated to the pool; no `balanceOf` calls in the hot trading path.
- **Library inlining:** `LSMath` is a `library` with `internal` functions. All math is inlined by the Solidity compiler with no cross-contract call overhead.
- **USDC address caching:** The USDC token address is fetched from the pool exactly once — during `initialize()` — and stored in the `usdc` slot. Every subsequent read (permit-path `buy()`, permit-path CSS `sell()`, `usdcToken()` view) is a single warm SLOAD with no external call overhead.
- **Canonical CEI in `buy()` and `sell()`:** Effects (storage writes) execute before the optional permit external call and before the pool interaction. State is fully committed prior to any external call; `nonReentrant` provides belt-and-suspenders protection and the pattern makes the security properties trivially auditable.

---

*Last updated: 2026. For developer implementation details see `BlieverMarket_DEV.md`.*
