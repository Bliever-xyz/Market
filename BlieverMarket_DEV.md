# BlieverMarket — Developer Implementation Reference

> **For:** Engineers analyzing, improving, debugging, or testing `BlieverMarket.sol`  
> **Prerequisites:** Read `BlieverMarket.md` (architecture) and `LSMath.sol` (mathematics)  
> **Solidity Version:** 0.8.31

---

## 1. File Map

| File | Role |
|---|---|
| `contracts/src/BlieverMarket.sol` | Main market contract — all logic described in this doc |
| `contracts/src/interfaces/IBlieverV1Pool.sol` | Minimal pool interface (4 calls + 1 view) |
| `contracts/src/LSMath.sol` | Stateless math library (imported, inlined) |
| `contracts/src/BlieverV1Pool.sol` | Central vault (separate, already deployed) |

---

## 2. Inheritance Chain

```
Initializable
    └── ReentrancyGuardTransient
            └── PausableUpgradeable
                    └── BlieverMarket
```

- **`Initializable`**: Provides `initializer` modifier and `_disableInitializers()`. Used because EIP-1167 clones cannot use constructors.
- **`ReentrancyGuardTransient`**: Provides `nonReentrant` modifier via EIP-1153 transient storage. Applied to `buy()`, `sell()`, `claim()` — any function that makes external calls after state changes. Because transient storage resets automatically at the end of every transaction, no storage slot is consumed and no initialiser call is required. Saves ~2,000 gas per guarded call compared to a persistent-storage guard.
- **`PausableUpgradeable`**: Provides `whenNotPaused` modifier. Applied to `buy()` and `sell()` only — resolution and claims are never paused.

> **Note:** `ReentrancyGuardTransient` is from the standard (non-upgradeable) OZ library; it carries no storage slots, so it is fully compatible with the EIP-1167 clone pattern without requiring a namespaced storage slot. `PausableUpgradeable` uses OZ's EIP-7201 namespaced storage. The market itself is NOT UUPS-upgradeable — there is no `_authorizeUpgrade()`.

---

## 3. Storage Layout Reference

**Critical:** The storage layout of `BlieverMarket` must **never be reordered**. Every deployed clone shares this layout via `DELEGATECALL`. Changing variable order breaks all existing markets.

`ReentrancyGuardTransient` uses EIP-1153 transient storage — it occupies **zero persistent storage slots**. `PausableUpgradeable` uses OZ's EIP-7201 namespaced storage (a keccak256-derived slot, not a sequential slot). As a result, `BlieverMarket`'s own variables occupy sequential slots starting at slot 0.

```solidity
// ── OZ Upgradeable Internal Storage (managed by OZ, EIP-7201 namespaced) ──
// PausableUpgradeable._paused stored at keccak256("openzeppelin.storage.Pausable") − 1
// ReentrancyGuardTransient: NO persistent storage — transient storage only (EIP-1153)

// ── BlieverMarket Variables (sequential, start at slot 0) ─────
// Packed Slot A: pool (20B) + outcomeCount (1B) + resolved (1B) + winningOutcome (1B) = 23B
address pool;               // 20 bytes
uint8   outcomeCount;       // 1 byte  [2–100]
bool    resolved;           // 1 byte
uint8   winningOutcome;     // 1 byte  [0–99]
                            // 9 bytes free in this slot

// Packed Slot B: resolver (20B) + tradingDeadline (5B) + resolutionDeadline (5B) = 30B
address resolver;           // 20 bytes
uint40  tradingDeadline;    // 5 bytes  [unix ts]
uint40  resolutionDeadline; // 5 bytes  [unix ts]
                            // 2 bytes free

// Slot C: factory (20 bytes, padded)
address factory;

// Slot D: questionId (32 bytes)
bytes32 questionId;

// Slot E: alpha (32 bytes)
uint256 alpha;

// Slot F: usdc (20 bytes, padded)
address usdc;

// Dynamic Array Slots: _quantities (length + data)
// Dynamic Array Slots: _initialQuantities (length + data)

// Mapping Slots: _shares[addr][idx]
// Mapping Slots: _totalTraderShares[idx]
// Mapping Slots: _claimed[addr]
```

**Invariant check:** When adding a new state variable, append it after `usdc` (before the dynamic arrays). Never insert between existing variables. Clones are immutable once deployed — new variables take effect only in clones deployed from an updated master.

---

## 4. Constants

```solidity
uint256 internal constant SHARE_TO_USDC    = 1e12;  // 10^(18−6)
uint256 internal constant MATH_SCALE       = 1e18;   // matches LSMath.SCALE
uint256 internal constant MIN_SHARE_AMOUNT = 1e15;   // 0.001 shares minimum (dust guard)
```

**SHARE_TO_USDC usage:**
| Operation | Rounding | Reason |
|---|---|---|
| Buy cost → USDC | **Ceiling** (`_ceilToUsdc`) | Vault collects at least cost; dust absorbed by LP |
| Sell refund → USDC | **Floor** (`_floorToUsdc`) | Vault pays out at most refund; dust stays in LP |
| Liability → USDC | **Floor** (`_floorToUsdc`) | Conservative: understates liability slightly |
| Claim payout → USDC | **Floor** (`_floorToUsdc`) | Sum of all payouts = settledPayout (epsilon excluded) |

**MIN_SHARE_AMOUNT rationale:**  
`buy()` and `sell()` both enforce `shareAmount >= MIN_SHARE_AMOUNT` (1e15, i.e. 0.001 shares in 18-dec). This prevents dust positions that would consume an SSTORE and emit an event for an economically negligible amount. At `MIN_SHARE_AMOUNT` the minimum meaningful on-chain position corresponds to a cost of approximately 0.001 USDC — a practical lower bound on Base L2 where gas costs are cheap enough to make sub-wei spam viable without this guard.

---

## 5. Modifiers

```solidity
modifier tradingOpen() {
    if (resolved || block.timestamp > tradingDeadline) revert TradingClosed();
    _;
}

modifier onlyResolver() {
    if (msg.sender != resolver) revert NotResolver();
    _;
}

modifier onlyFactory() {
    if (msg.sender != factory) revert NotFactory();
    _;
}
```

Note that `tradingOpen` checks BOTH `resolved` (boolean) AND `tradingDeadline` (timestamp). A market can stop trading either when the oracle resolves early OR when the deadline passes — whichever comes first.

---

## 6. `initialize()` — Deep Dive

**Caller:** `BlieverMarketFactory` (immediately after deploying the clone).  
**Called:** Exactly once per clone (enforced by OZ `initializer`).

### Parameters

| Parameter | Type | Notes |
|---|---|---|
| `_pool` | `address` | `BlieverV1Pool` proxy address |
| `_questionId` | `bytes32` | UMA oracle question ID, keccak256 of ancillary data |
| `_nOutcomes` | `uint8` | [2, 100] — validated against `LSMath.MAX_OUTCOMES` |
| `_alpha` | `uint256` | 18-dec, [1e12, 2e17] — validated by `LSMath.validateAlpha()` |
| `_tradingDeadline` | `uint40` | Must be > `block.timestamp` |
| `_resolutionDeadline` | `uint40` | Must be > `_tradingDeadline` |
| `_epsilon` | `uint256` | Per-outcome initial seed (18-dec) — computed by factory off-chain |
| `_resolver` | `address` | Resolution Adapter address |
| `_factory` | `address` | Factory address (admin rights) |

### Epsilon Validation

The `initialize()` function calls `LSMath.liquidityParameter(initQ, _alpha)` with the seeded q-vector. This validates that:
- The q-vector is non-empty
- All quantities are ≥ 0 (trivially true for `uint256`)
- Sum of quantities > 0 (would revert on `ZeroQuantitySum` if epsilon = 0, but epsilon is separately checked)
- alpha is within range (redundant with direct check, belt-and-suspenders)

### Why Two Copies of initQ?

```solidity
_quantities        = initQ;
_initialQuantities = initQ;
```

`_quantities` is mutated by every trade. `_initialQuantities` is **never mutated after initialization** — it serves as the source of truth for `getInitialQuantities()` view calls. Its derivative `_initialCost = C(q⁰)` is computed once at initialization and stored in a dedicated slot; every buy and sell reads `_initialCost` via a single warm SLOAD to feed `calculateWorstCaseLossFromCosts()`, replacing any need to reload `_initialQuantities` in the trading hot path.

Additionally, `initialize()` calls `IBlieverV1Pool(_pool).asset()` exactly once and stores the result in the `usdc` slot. This is the only point in the contract's lifetime where the USDC address is fetched externally — subsequent reads in `buy()` (permit path) and `usdcToken()` read the cached slot with a single warm SLOAD.

---

## 7. `buy()` — Function Flow

```
buy(outcomeIndex, shareAmount, maxCostUsdc, deadline, v, r, s)
│
├── MODIFIERS: nonReentrant, whenNotPaused, tradingOpen
│
├── CHECKS
│   ├── shareAmount >= MIN_SHARE_AMOUNT (1e15)  — dust guard
│   └── outcomeIndex < outcomeCount
│
├── LOAD STATE (single combined loop)
│   ├── n = outcomeCount (cache: avoid repeated SLOAD)
│   ├── _alpha = alpha (cache)
│   └── (qOld, qNew) = _loadQuantitiesForBuy(n, outcomeIndex, shareAmount)
│       └── One pass: reads all n SLOADs, builds qOld and qNew simultaneously
│           with qNew[outcomeIndex] = qOld[outcomeIndex] + shareAmount
│
├── MATH
│   ├── (tradeCost18, costNew) = LSMath.calculateTradeCostDetailed(qOld, qNew, _alpha)
│   │   └── Returns both C(qNew)−C(qOld) [int256] AND C(qNew) [uint256] in one pass.
│   │       costFunction(qNew) is evaluated exactly once — costNew is reused below.
│   ├── assert tradeCost18 >= 0 (revert NegativeBuyCost if not)
│   ├── costUsdc = _ceilToUsdc(uint256(tradeCost18))
│   ├── newLiability18 = LSMath.calculateWorstCaseLossFromCosts(costNew, _initialCost, qNew)
│   │   └── _initialCost = C(q⁰) read from single warm SLOAD (cached at initialize()).
│   │       No _loadInitialQuantities() call; no redundant costFunction(q⁰) evaluation.
│   └── newLiabilityUsdc = _floorToUsdc(newLiability18)
│
├── SLIPPAGE CHECK
│   └── if costUsdc > maxCostUsdc: revert SlippageExceeded
│
├── EFFECTS (CEI — all SSTOREs committed before any external call)
│   ├── _quantities[outcomeIndex] += shareAmount  [1 SSTORE — single slot only]
│   ├── _shares[msg.sender][outcomeIndex] += shareAmount [1 SLOAD + 1 SSTORE]
│   └── _totalTraderShares[outcomeIndex] += shareAmount  [1 SLOAD + 1 SSTORE]
│
├── OPTIONAL PERMIT (v != 0 only — external call after state committed)
│   ├── usdc read from cached slot  [1 warm SLOAD — no external call to pool]
│   ├── try IERC20Permit(usdc).permit(msg.sender, pool, maxCostUsdc, deadline, v, r, s)
│   └── catch: if allowance(msg.sender, pool) < maxCostUsdc → revert InsufficientPermitAllowance
│       └── Silent fallback on consumed nonce — front-run griefing does not brick the trade
│
└── INTERACTION
    └── IBlieverV1Pool(pool).collectTradeCost(msg.sender, costUsdc, newLiabilityUsdc)
        └── Pool: IERC20(usdc).safeTransferFrom(trader, pool, costUsdc)
            ⚠️  Trader must have approved pool (not this contract) for ≥ maxCostUsdc USDC,
                OR supply a valid EIP-2612 permit signature via v/r/s/deadline.
                Pass v=0/r=0/s=0/deadline=0 to skip permit and rely on pre-existing approval.
```

**Gas profile estimate** (n=2 outcomes): ~85,000–115,000 gas on Base  
- `_loadQuantitiesForBuy`: 2 cold SLOADs (one combined loop, no copy pass) = ~4,200 gas  
- `calculateTradeCostDetailed`: runs `costFunction(qNew)` exactly once — returns both `tradeCost18` and `costNew` in a single pass (no second `costFunction` call in the liability step)  
- `calculateWorstCaseLossFromCosts`: reads `_initialCost` via 1 warm SLOAD (100 gas); runs O(n) scan for `max(qᵢ)` — no `exp`/`ln` at this step  
- 1 SSTORE to `_quantities[outcomeIndex]` (warm) = ~100 gas vs. n×SSTORE previously  
- Permit path (when used): 1 warm SLOAD for `usdc` — no external call to pool in this path  
- 1 `collectTradeCost` external call + `safeTransferFrom` USDC: ~25,000 gas  

---

## 8. `sell()` — Function Flow with CSS

The sell function is the most complex. Follow this annotated flow carefully.

```
sell(outcomeIndex, shareAmount, minRefundUsdc, maxCostUsdc, deadline, v, r, s)
│
├── MODIFIERS: nonReentrant, whenNotPaused, tradingOpen
│
├── CHECKS
│   ├── shareAmount >= MIN_SHARE_AMOUNT (1e15)  — dust guard
│   └── outcomeIndex < outcomeCount
│
├── LOAD STATE (single targeted read — not the full array)
│   ├── trader = msg.sender
│   ├── n, _alpha (cached)
│   └── traderBal = _shares[trader][outcomeIndex]   [1 mapping SLOAD]
│       Unlike buy(), only the single sold-outcome balance is read here.
│       The remaining n−1 outcome slots are only written (+tBar) in the
│       effects step — they are never read in the cost computation path.
│
├── CSS TRANSLATION COMPUTATION
│   │
│   ├── tBar = max(0, shareAmount − traderBal)
│   │         ↑ 0 if trader has enough; positive if CSS applies
│   │
│   └── netReduce = shareAmount − tBar
│                   ↑ how much q[outcomeIndex] decreases in the market's q-vector
│
├── BUILD q_old and q_new (single combined loop)
│   └── (qOld, qNew) = _loadQuantitiesForSell(n, outcomeIndex, netReduce, tBar)
│       One pass: reads all n quantity SLOADs, builds qOld and qNew simultaneously.
│         • qNew[outcomeIndex] = qOld[outcomeIndex] − netReduce
│           (actual_delta[sold] = −shareAmount + tBar = −netReduce; tBar is NOT re-added)
│         • qNew[i ≠ outcomeIndex] = qOld[i] + tBar
│       InsufficientMarketQuantity guard applied inline at the idx slot.
│       Replaces: _loadQuantities(n) + _copyArray(qOld, n) + CSS mutation loop.
│
├── MATH
│   ├── (tradeCost18, costNew) = LSMath.calculateTradeCostDetailed(qOld, qNew, _alpha)
│   │   └── Returns C(qNew)−C(qOld) [int256, can be negative] AND C(qNew) in one pass.
│   ├── newLiability18 = LSMath.calculateWorstCaseLossFromCosts(costNew, _initialCost, qNew)
│   │   └── _initialCost read from 1 warm SLOAD; no _loadInitialQuantities(n) call.
│   ├── newLiabilityUsdc = _floorToUsdc(newLiability18)
│   │
│   ├── isRefund = (tradeCost18 < 0)
│   ├── absAmount18 = |tradeCost18|
│   └── absAmountUsdc = isRefund ? _floorToUsdc : _ceilToUsdc [direction matters]
│
├── SLIPPAGE CHECKS (bidirectional)
│   ├── if isRefund  && absAmountUsdc < minRefundUsdc → revert SlippageExceeded
│   └── if !isRefund && absAmountUsdc > maxCostUsdc   → revert SlippageExceeded
│
├── EFFECTS (CEI: all SSTOREs before external call)
│   ├── _storeQuantities(qNew, n) [n SSTOREs]
│   │
│   ├── UPDATE SOLD OUTCOME
│   │   new position = traderBal + tBar − shareAmount   (proven ≥ 0 by CSS invariant)
│   │   unchecked: _shares[trader][outcomeIndex]    = traderBal + tBar − shareAmount
│   │   unchecked: _totalTraderShares[outcomeIndex] += tBar − shareAmount
│   │
│   └── if tBar > 0: for all j ≠ outcomeIndex:
│           _shares[trader][j] += tBar
│           _totalTraderShares[j] += tBar
│
└── INTERACTION (direction depends on isRefund)
    ├── if isRefund:
    │     pool.distributeRefund(trader, absAmountUsdc, newLiabilityUsdc)
    │     Pool: IERC20(usdc).safeTransfer(trader, absAmountUsdc)
    │     ← No permit ever attempted on the refund path (gas saved)
    │
    └── else (CSS net-cost):
          ── OPTIONAL PERMIT (v != 0 only — gated inside !isRefund) ──
          ├── usdc read from cached slot  [1 warm SLOAD]
          ├── try IERC20Permit(usdc).permit(trader, pool, maxCostUsdc, deadline, v, r, s)
          └── catch: if allowance(trader, pool) < absAmountUsdc → revert InsufficientPermitAllowance
              └── Silent fallback on consumed nonce — front-run griefing does not brick the trade

          pool.collectTradeCost(trader, absAmountUsdc, newLiabilityUsdc)
          Pool: IERC20(usdc).safeTransferFrom(trader, pool, absAmountUsdc)
          ⚠️  Trader must have approved pool (not this contract) for ≥ maxCostUsdc USDC,
              OR supply a valid EIP-2612 permit signature via v/r/s/deadline.
              Pass v=0/r=0/s=0/deadline=0 to skip permit and rely on pre-existing approval.
```

### CSS Arithmetic Proof (Sold Outcome New Balance)

Let:
- `B = traderBal` (trader's old balance for sold outcome — single SLOAD)
- `S = shareAmount` (requested sell amount)
- `T = tBar = max(0, S − B)`
- `R = netReduce = S − T`

**CSS q-vector delta (Othman et al. §3.3.2):**

```
actual_delta[i=sold] = −S + T = −R         (= −netReduce)
actual_delta[j≠i]    = +T                   (= +tBar)
```

This means tBar is the translation added to every OTHER outcome's q slot. For the sold outcome, the entire net delta is already captured by `−netReduce` — adding tBar again would overstate q[sold] by tBar units, breaking the market state invariant.

**Trader balance change:**

New position = B + actual_delta[i] = B − R = B + T − S

Case A: T = 0 (B ≥ S):  
`new = B − S ≥ 0` ✓ (trader had enough shares)

Case B: T = S − B (B < S):  
`new = B − S + (S − B) = 0` ✓ (trader sold everything, position goes to zero)

In both cases: **new position ≥ 0**. CSS invariant maintained. ✓  
Because underflow cannot occur, both balance writes use `unchecked` blocks.

**Concrete verification (Scenario: CSS triggers):**

q = [1100, 900], traderBal = 300, shareAmount = 500, tBar = 200, netReduce = 300.

- Correct: qNew[0] = 1100 − 300 = **800** (`q − netReduce`)
- Buggy:   qNew[0] = 1100 − 300 + 200 = **1000** (`q − netReduce + tBar` — inflated by tBar)

Sum-of-positions check: trader gets [0, 200], epsilon [100, 100], others [700, 800].  
q[0] = 0+100+700 = **800** ✓ matches correct formula only.

### `_totalTraderShares` update for sold outcome

```solidity
unchecked {
    _totalTraderShares[outcomeIndex] = _totalTraderShares[outcomeIndex] + tBar - shareAmount;
}
```

This is equivalent to `+= (tBar − shareAmount) = −netReduce`. When tBar = 0, it's a simple decrease by shareAmount. When tBar > 0 (CSS), the decrease is by `netReduce = shareAmount − tBar = B` (the actual position sold).

**Underflow guard:** `_totalTraderShares[i]` is the sum of all traders' shares for outcome i. Since `netReduce = shareAmount − tBar ≤ traderBal ≤ _totalTraderShares[outcomeIndex]`, underflow cannot occur. The `unchecked` block is safe by this invariant.

**Gas profile estimate** (n=2 outcomes, standard refund sell): ~95,000–130,000 gas on Base  
- `_shares[trader][outcomeIndex]`: 1 cold mapping SLOAD = ~2,100 gas (replaces n SLOADs from `_loadTraderShares`)  
- `_loadQuantitiesForSell`: 2 cold SLOADs in one combined loop (replaces `_loadQuantities` + `_copyArray` + CSS loop = 3 passes)  
- `calculateTradeCostDetailed`: runs `costFunction(qNew)` exactly once — returns both `tradeCost18` and `costNew` in a single pass  
- `calculateWorstCaseLossFromCosts`: 1 warm SLOAD for `_initialCost`; O(n) scan — no `exp`/`ln`  
- `_storeQuantities`: n SSTOREs (sell always writes full q-vector; CSS may change every slot)  
- Permit path: **never reached** on standard refund sells — zero overhead on the common path  
- 1 `distributeRefund` external call + `safeTransfer` USDC: ~25,000 gas

---

## 9. `resolve()` — Function Flow

```
resolve(uint8 _winningOutcome)
│
├── MODIFIER: onlyResolver
│
├── CHECKS
│   ├── !resolved (not already resolved)
│   ├── block.timestamp <= resolutionDeadline (resolver was timely)
│   └── _winningOutcome < outcomeCount (valid index)
│
├── EFFECTS
│   ├── resolved = true       [1 SSTORE — writes into packed slot A]
│   └── winningOutcome = W    [packed into slot A, same SSTORE potentially]
│
├── COMPUTE PAYOUT
│   └── totalPayoutUsdc = _floorToUsdc(_totalTraderShares[W])
│                         ↑ floor division → EXCLUDES epsilon dust
│                         ↑ result = exact USDC claimable by all winning traders
│
└── INTERACTION
    └── IBlieverV1Pool(pool).settleMarket(totalPayoutUsdc)
        Pool effects:
          info.settled = true
          info.settledPayout = totalPayoutUsdc
          info.currentLiability = 0  [releases from totalLiability]
          totalLiability -= liveLiab
          --activeMarketCount
        Pool emits: MarketSettled(market, totalPayout, profit)
```

**Why is resolution NOT `nonReentrant`?**  
`resolve()` is called by the trusted `resolver` address (the Resolution Adapter), which is set immutably at initialization. The pool's `settleMarket()` is already `nonReentrant` on the pool side. Adding `nonReentrant` here would consume a transient storage write/read for a trusted caller scenario where re-entry is not a realistic attack vector. The CEI pattern is still followed (effects before interaction).

**Why is resolution NOT `whenNotPaused`?**  
Pausing the market should never block resolution. Even if the factory pauses trading during an oracle dispute, the market must be settleable once the oracle finalises. This mirrors `BlieverV1Pool.settleMarket()` which is also not pause-gated.

---

## 10. `claim()` — Function Flow

```
claim()
│
├── MODIFIER: nonReentrant
│
├── CHECKS
│   ├── resolved == true
│   ├── !_claimed[caller]
│   └── _shares[caller][winningOutcome] > 0
│
├── DUST PATH (payoutUsdc == 0 after floor conversion)
│   │   Shares < SHARE_TO_USDC (1e12 units, i.e. < 1 USDC-wei).
│   │   Does NOT revert. Marks position processed and returns.
│   ├── _claimed[caller] = true          [1 SSTORE — prevents repeat attempts]
│   ├── emit DustForfeited(caller, wo, shares18)
│   └── return  ← pool.claimWinnings is NOT called (amount = 0 would revert there)
│
├── NORMAL PATH (payoutUsdc > 0)
│   │
│   ├── EFFECTS (before interaction — CEI)
│   │   └── _claimed[caller] = true  [1 SSTORE]
│   │       ↑ CRITICAL: set before external call to prevent reentrancy double-claim
│   │
│   └── INTERACTION
│       └── IBlieverV1Pool(pool).claimWinnings(caller, payoutUsdc)
│           Pool effects:
│             info.claimedPayout += payoutUsdc
│             safeTransfer(caller, payoutUsdc)
│             if claimedPayout == settledPayout:
│               _revokeRole(MARKET_ROLE, market)
│               emit MarketFullyClaimed(...)
```

**Reentrancy note:** The `_claimed[caller] = true` SSTORE happens before the pool call on the normal path, and before the `return` on the dust path. Either way, any re-entry into `claim()` by the caller's `receive()` function hits `AlreadyClaimed` immediately. `nonReentrant` (via `ReentrancyGuardTransient`) provides belt-and-suspenders protection via transient storage.

**Dust path — why emit rather than revert:** A revert on dust would leave the caller unable to mark their position as processed and unable to know whether to retry. The `DustForfeited` event provides a clean, observable on-chain signal to the caller and to off-chain indexers. The dust value is already excluded from `pool.settledPayout` (via the same floor division in `resolve()`), so no USDC is locked — it is absorbed into LP NAV as part of the market-making operating cost. The `_claimed` flag prevents repeated failed attempts from polluting the chain.

---

## 11. `expireUnresolved()` — Factory Safety Hatch

```
expireUnresolved()
│
├── MODIFIER: onlyFactory
│
├── CHECKS
│   ├── !resolved (market not already resolved)
│   └── block.timestamp > resolutionDeadline (deadline passed)
│
├── EFFECTS
│   └── resolved = true
│       (winningOutcome stays 0; no legitimate claim will work since settledPayout=0)
│
├── INTERACTION
│   └── pool.settleMarket(0)
│       Pool: settledPayout = 0
│             currentLiability = 0 (liability released, vault absorbs as loss)
│             --activeMarketCount
│             if totalPayout == 0: _revokeRole(MARKET_ROLE, market) immediately
│
└── EVENT
    └── emit MarketExpired(factory, uint40(block.timestamp), ExpiryReason.TIMEOUT)
        ├── factory  — the caller (onlyFactory enforced above)
        ├── timestamp — block.timestamp cast to uint40
        └── ExpiryReason.TIMEOUT — resolutionDeadline passed with no oracle resolution
            Off-chain indexers should handle unknown ExpiryReason values gracefully;
            the enum is reserved for future oracle-error classification without ABI breakage.
```

**No traders are harmed:** USDC traders paid during buy trades was already collected by the pool during active trading. The vault absorbs the `riskBudget` as a loss (the LS-LMSR worst-case guarantee). `settleMarket(0)` means `MARKET_ROLE` is immediately revoked (pool logic: zero-payout revocation path).

---

## 12. View Functions — Quick Reference

| Function | Gas Estimate | Notes |
|---|---|---|
| `getPrice(i)` | ~15,000 | Calls `LSMath.getPrice()` → `getAllPrices()` |
| `getAllPrices()` | ~15,000 | Single pass, reuses intermediate values |
| `getBuyCost(i, amt)` | ~12,000 | Computes `calculateTradeCost` for quote |
| `getSellEstimate(trader, i, amt)` | ~18,000 | Includes CSS logic + `calculateTradeCost` |
| `getCssTranslation(trader, i, amt)` | ~2,000 | Single mapping read |
| `getShares(trader, i)` | ~2,000 | Single mapping read |
| `getAllShares(trader)` | ~2,000 × n | n mapping reads |
| `getQuantities()` | ~2,000 × n | n storage reads |
| `getMarketStatus()` | ~4,000 × n | n reads + sum |
| `getSumOfPrices()` | ~15,000 | Calls `LSMath.sumOfPrices()` |

---

## 13. Internal Helper Functions

### `_loadQuantities(n)` / `_loadInitialQuantities(n)`
```solidity
function _loadQuantities(uint256 n) internal view returns (uint256[] memory q) {
    q = new uint256[](n);
    for (uint256 i = 0; i < n;) {
        q[i] = _quantities[i];   // cold SLOAD first time, warm after
        unchecked { ++i; }
    }
}
```
Called at the start of `sell()` to snapshot the current quantity vector into memory. Also used by view functions (`getQuantities`, `getMarketStatus`). Not used by `buy()` — the latter uses `_loadQuantitiesForBuy` instead.

### `_loadQuantitiesForBuy(n, idx, delta)`
```solidity
function _loadQuantitiesForBuy(uint256 n, uint256 idx, uint256 delta)
    internal view
    returns (uint256[] memory qOld, uint256[] memory qNew)
{
    qOld = new uint256[](n);
    qNew = new uint256[](n);
    for (uint256 i = 0; i < n;) {
        uint256 q = _quantities[i];
        qOld[i] = q;
        qNew[i] = (i == idx) ? q + delta : q;
        unchecked { ++i; }
    }
}
```
Reads all n storage slots once and simultaneously builds both `qOld` (unmodified snapshot) and `qNew` (with `qNew[idx] += delta`). Replaces the pattern of `_loadQuantities(n)` followed by `_copyArray(qOld, n)` — two full memory passes reduced to one. Called exclusively by `buy()`.

### `_loadQuantitiesForSell(n, idx, netReduce, tBar)`
```solidity
function _loadQuantitiesForSell(uint256 n, uint256 idx, uint256 netReduce, uint256 tBar)
    internal view
    returns (uint256[] memory qOld, uint256[] memory qNew)
{
    qOld = new uint256[](n);
    qNew = new uint256[](n);
    for (uint256 i = 0; i < n;) {
        uint256 q = _quantities[i];
        qOld[i] = q;
        if (i == idx) {
            if (netReduce > 0 && q < netReduce) revert InsufficientMarketQuantity();
            unchecked { qNew[i] = q - netReduce; }   // actual_delta[sold] = −netReduce
        } else {
            qNew[i] = q + tBar;                       // actual_delta[j≠i] = +tBar
        }
        unchecked { ++i; }
    }
}
```
Mirrors `_loadQuantitiesForBuy` for the sell path. Reads all n storage slots once and simultaneously constructs `qOld` and the CSS-translated `qNew` in a single loop pass. Replaces the three-step pattern of `_loadQuantities(n)` + `_copyArray(qOld, n)` + a separate CSS mutation loop.

**Critical CSS q-vector formula:** `qNew[idx] = qOld[idx] − netReduce`. The `tBar` translation is applied only to `j ≠ idx`. For the sold outcome, `netReduce = shareAmount − tBar` fully encodes the net delta (`actual_delta[sold] = −shareAmount + tBar = −netReduce`). Adding `+tBar` to the sold outcome slot would overstate it by `tBar` units, inflating the market quantity vector and corrupting all subsequent cost and liability calculations.

The `InsufficientMarketQuantity` guard is embedded at the `idx` slot — no separate pre-check loop is needed. Called exclusively by `sell()`.

### `_loadTraderShares(trader, n)`
```solidity
function _loadTraderShares(address trader, uint256 n) internal view returns (uint256[] memory qt) {
    qt = new uint256[](n);
    for (uint256 i = 0; i < n;) {
        qt[i] = _shares[trader][i];   // mapping SLOAD: slot = keccak(trader, keccak(i, slot))
        unchecked { ++i; }
    }
}
```
Loads all n per-outcome balances into a memory array. Used in **view functions only** — `getSellEstimate()` and `getAllShares()` — where the full position vector is needed to simulate CSS without state mutations. The `sell()` write path reads only `_shares[trader][outcomeIndex]` directly (a single mapping SLOAD) to compute `tBar`, avoiding the n-SLOAD overhead of this helper on every live sell transaction.

### `_storeQuantities(qNew, n)`
```solidity
function _storeQuantities(uint256[] memory qNew, uint256 n) internal {
    for (uint256 i = 0; i < n;) {
        _quantities[i] = qNew[i];   // SSTORE (warm after _loadQuantitiesForSell)
        unchecked { ++i; }
    }
}
```
Writes the full mutated quantity vector back to storage. Called by `sell()` (CEI-compliant, before external call). Not called by `buy()` — buys write only the single changed slot directly (`_quantities[outcomeIndex] += shareAmount`). For `sell()`, the full write-back is necessary because CSS may increment every slot in `qNew`.

### `_floorToUsdc(amount18)` / `_ceilToUsdc(amount18)`
```solidity
function _floorToUsdc(uint256 amount18) internal pure returns (uint256) {
    return amount18 / SHARE_TO_USDC;                              // floor (truncate)
}

function _ceilToUsdc(uint256 amount18) internal pure returns (uint256) {
    return (amount18 + SHARE_TO_USDC - 1) / SHARE_TO_USDC;       // ceiling
}
```
Both are `pure` functions with no overflow risk at realistic values: `amount18` would need to be `> type(uint256).max − 1e12` for `_ceilToUsdc` to overflow, which is impossible with LS-LMSR amounts.

### `_copyArray(src, n)`
```solidity
function _copyArray(uint256[] memory src, uint256 n) internal pure returns (uint256[] memory dst) {
    dst = new uint256[](n);
    for (uint256 i = 0; i < n;) {
        dst[i] = src[i];
        unchecked { ++i; }
    }
}
```
Deep copy of a memory array. Used exclusively by **view functions** (`getBuyCost`, `getSellEstimate`) where `qOld` must remain unmodified alongside a separately mutated `qNew`. Not used in the `buy()` or `sell()` write paths — `_loadQuantitiesForBuy` and `_loadQuantitiesForSell` each build both arrays in a single storage-read pass, making a post-load copy unnecessary.

---

## 14. Error Reference

| Error | Trigger Condition |
|---|---|
| `NotResolver()` | `msg.sender != resolver` in `resolve()` |
| `NotFactory()` | `msg.sender != factory` in `pause()`, `unpause()`, `expireUnresolved()` |
| `ZeroAddress()` | Zero address in `initialize()` parameters |
| `ZeroAmount()` | Reserved — no longer used in trading path (see `ShareAmountTooSmall`) |
| `ZeroEpsilon()` | `_epsilon == 0` in `initialize()` |
| `InvalidOutcomeIndex(index, max)` | `outcomeIndex >= outcomeCount` |
| `InvalidOutcomeCount(count)` | `_nOutcomes < 2` or `> 100` in `initialize()` |
| `InvalidAlpha(value)` | Alpha outside `[1e12, 2e17]` (LSMath bounds) |
| `InvalidDeadlines()` | Deadlines in the past, or `tradingDeadline >= resolutionDeadline` |
| `TradingClosed()` | `resolved == true` or `block.timestamp > tradingDeadline` |
| `MarketAlreadyResolved()` | `resolve()` called twice, or `expireUnresolved()` after resolution |
| `MarketNotResolved()` | `claim()` called before resolution |
| `AlreadyClaimed()` | `claim()` called by same address twice |
| `NoWinningShares()` | Caller has 0 shares of the winning outcome |
| `ShareAmountTooSmall(amount, minimum)` | `shareAmount < MIN_SHARE_AMOUNT` in `buy()` or `sell()` |
| `SlippageExceeded(actual, limit)` | Cost > `maxCostUsdc` (buy), refund < `minRefundUsdc` (sell refund path), or CSS net cost > `maxCostUsdc` (sell cost path) |
| `ResolutionDeadlinePassed()` | `resolve()` called after `resolutionDeadline` |
| `ResolutionDeadlineNotPassed()` | `expireUnresolved()` called before deadline passes |
| `InsufficientMarketQuantity()` | `qOld[outcomeIndex] < netReduce` (should never trigger if ledger invariant holds) |
| `NegativeBuyCost()` | `calculateTradeCost` returned negative for a buy (LSMath bug — should never happen) |
| `InsufficientPermitAllowance()` | Permit signature was consumed (front-run) and existing allowance is below `maxCostUsdc` |

**Events associated with terminal claim outcomes:**

| Event | Condition |
|---|---|
| `WinningsClaimed(winner, wo, shares18, payoutUsdc)` | Normal path: `payoutUsdc > 0`; USDC transferred from vault to winner |
| `DustForfeited(winner, wo, shares18)` | Dust path: `payoutUsdc == 0`; position marked processed, no USDC transferred |

---

## 15. Key Invariants to Verify in Tests

These should hold after every state transition:

| # | Invariant | How to Check |
|---|---|---|
| I1 | `_quantities[i] >= epsilon` for all i, before any sells | Read `_quantities[]` after initialization |
| I2 | `Σ _shares[any_trader][i] <= _quantities[i] - epsilon_i` for all i | Aggregate across traders |
| I3 | `_totalTraderShares[i] == Σ _shares[all_traders][i]` | Sum all trader shares, compare to mapping |
| I4 | `_quantities[i] >= 0` for all i after all trades | Never should underflow (CSS guarantee) |
| I5 | After resolution: `_totalTraderShares[winningOutcome] / SHARE_TO_USDC == pool.settledPayout` | Read both |
| I6 | After claim: `_claimed[trader] == true` | Read mapping |
| I7 | `_totalTraderShares[i] <= _quantities[i]` for all i | Trader shares never exceed market quantity |
| I8 | Sum of `newLiability` across all pool.collectTradeCost calls ≤ `riskBudget` | Pool view `getMarketInfo()` |
| I9 | After a CSS sell: `_quantities[sold] = _quantities_before[sold] − netReduce` (NOT `− netReduce + tBar`) | Read q-vector before/after CSS sell; confirm `q[sold]` decreased by exactly `netReduce = shareAmount − tBar` |

---

## 16. Common Debugging Scenarios

**Problem:** `buy()` reverts with `SlippageExceeded`.  
**Cause:** Another trade changed the q-vector between the UI quote and the on-chain execution.  
**Fix:** Increase `maxCostUsdc` by 1–5% for slippage tolerance, or re-quote immediately before submitting.

**Problem:** `buy()` or `sell()` reverts with `ShareAmountTooSmall`.  
**Cause:** `shareAmount` is below `MIN_SHARE_AMOUNT` (1e15). The position is economically negligible and is rejected to prevent state pollution.  
**Fix:** Enforce `shareAmount >= 1e15` in the frontend before submitting. For context, 1e15 = 0.001 shares = approximately $0.001 USDC at a 50% price.

**Problem:** `buy()` reverts with `InsufficientPermitAllowance`.  
**Cause:** A permit signature was supplied (`v != 0`), but the nonce was already consumed — most likely by a front-running bot that extracted the permit from the mempool. The fallback allowance check found the trader's allowance for the pool is below `maxCostUsdc`.  
**Fix (immediate):** Re-submit with `v=0, r=0, s=0, deadline=0` and a prior `USDC.approve(pool, amount)` call. The permit path is optional and does not need to be retried. **Fix (long-term):** Frontend should use `deadline` values close to the current block timestamp to limit the griefing window.

**Problem:** After a CSS sell, `getQuantities()[outcomeIndex]` does not match the expected value: `q_before[i] − netReduce`.  
**Cause:** A prior version of the contract incorrectly applied `+tBar` to the sold outcome slot in the q-vector (equivalent to computing `q − netReduce + tBar` instead of `q − netReduce`). This is the CSS q-vector bug.  
**Diagnosis:** Compute `expected = q_before[sold] − (shareAmount − tBar)`. If actual `q[sold] > expected` by exactly `tBar`, the buggy formula was used.  
**Impact:** All trades after a CSS sell are priced off an inflated q-vector. The sold outcome's price is understated, and the overall liability is incorrectly computed.  
**Fix:** Confirmed corrected in current code — `_loadQuantitiesForSell` uses `q - netReduce` for the idx slot (not `q - netReduce + tBar`). The `getSellEstimate` view function applies the same fix.

**Problem:** `sell()` reverts with `TradingClosed`.  
**Cause:** Either `block.timestamp > tradingDeadline` or `resolved == true`.  
**Fix:** Check both conditions with `getMarketStatus()`.

**Problem:** `sell()` reverts with `SlippageExceeded` on the refund side.  
**Cause:** Another trade changed the q-vector between the UI quote and the on-chain execution, causing the refund to fall below `minRefundUsdc`.  
**Fix:** Increase slippage tolerance (decrease `minRefundUsdc` by 1–5%) or re-quote immediately before submitting.

**Problem:** `sell()` reverts with `SlippageExceeded` on a CSS net-cost sell.  
**Cause:** The CSS translation produced a net payment to the vault that exceeds `maxCostUsdc`. This is an unexpected cost scenario — either market conditions shifted or the CSS translation was larger than anticipated.  
**Fix:** Re-quote via `getSellEstimate()`. If CSS is unexpectedly large, the trader may need to split the sell into smaller amounts. Pass a higher `maxCostUsdc` only after understanding the CSS cost.

**Problem:** `sell()` reverts with `InsufficientPermitAllowance` on a CSS net-cost sell.  
**Cause:** A permit signature was supplied (`v != 0`) but the nonce was already consumed, and the trader's existing allowance for the pool is below `absAmountUsdc`.  
**Fix (immediate):** Re-submit with `v=0, r=0, s=0, deadline=0` and a prior `USDC.approve(pool, maxCostUsdc)` call.  
**Fix (long-term):** Use a `deadline` close to the current block timestamp to narrow the griefing window.

**Problem:** `claim()` reverts with `NoWinningShares`.  
**Cause:** Caller holds 0 shares for the winning outcome. They may have held a different outcome.  
**Fix:** Call `getShares(caller, winningOutcome)` before attempting claim.

**Problem:** `claim()` succeeds but emits `DustForfeited` instead of `WinningsClaimed`.  
**Cause:** Caller holds winning shares but fewer than `SHARE_TO_USDC` (1e12 units, i.e. < 1 USDC-wei). Floor conversion yields 0. The position is economically negligible (< $0.000001 USDC) and is marked as processed without a USDC transfer.  
**Effect:** `_claimed[caller]` is set to `true`. No funds are locked — the dust value was already excluded from `pool.settledPayout` via the same floor division in `resolve()` and is absorbed into LP NAV.  
**Fix:** No action required. The position has been fully processed.

**Problem:** `resolve()` reverts with `ResolutionDeadlinePassed`.  
**Cause:** Oracle took too long; `block.timestamp > resolutionDeadline`.  
**Fix:** Factory should call `expireUnresolved()` instead.

**Problem:** After all traders claim, `pool.markets[market].claimedPayout < settledPayout`.  
**Cause:** Individual `_floorToUsdc` calls may accumulate tiny rounding errors.  
**Diagnosis:** Check difference: `settledPayout - claimedPayout`. Should be < 1 USDC per trader.  
**Fix (if needed):** If this prevents `MARKET_ROLE` revocation, pool's governance can call `forceSettleMarket()`.

---

## 17. Integration Checklist for Factory / Adapter Developers

**When deploying a clone:**
- [ ] Compute `epsilon` off-chain: `ε_18dec = (maxRiskPerMarket * 1e12) / (1 + alpha * n * ln(n) / 1e18)` (all 18-dec math)
- [ ] Call `pool.registerMarket(cloneAddress, nOutcomes)` BEFORE calling `clone.initialize()`
  - Pool grants `MARKET_ROLE` to `cloneAddress` at this point
- [ ] Call `clone.initialize(pool, questionId, nOutcomes, alpha, tradingDeadline, resolutionDeadline, epsilon, resolver, factory)`
- [ ] Verify: `pool.isActiveMarket(cloneAddress) == true`

**When resolving:**
- [ ] Resolver calls `market.resolve(winningOutcome)`
- [ ] Check emitted `MarketResolved` event for `totalPayoutUsdc`
- [ ] Verify: `pool.getMarketInfo(market).settled == true`

**When expiring (oracle failure):**
- [ ] Verify: `block.timestamp > market.resolutionDeadline()`
- [ ] Factory calls `market.expireUnresolved()`
- [ ] Verify: `market.resolved() == true` and `pool.getMarketInfo(market).settledPayout == 0`
- [ ] Check emitted `MarketExpired(factory, timestamp, ExpiryReason.TIMEOUT)` event — `reason` field is always `TIMEOUT` for deadline expiry; reserved for future oracle-error paths without ABI breakage

---

## 18. IBlieverV1Pool Interface — Contract Connection Points

The interface in `interfaces/IBlieverV1Pool.sol` is the only connection between `BlieverMarket` and the vault. Four write functions + one view:

```solidity
function collectTradeCost(address trader, uint256 cost, uint256 newLiability) external;
// Called by: buy(), and sell() when CSS causes net payment
// Effect: safeTransferFrom(trader, pool, cost); updates currentLiability

function distributeRefund(address trader, uint256 refundAmount, uint256 newLiability) external;
// Called by: sell() when net refund
// Effect: safeTransfer(trader, refundAmount); updates currentLiability

function settleMarket(uint256 totalPayout) external;
// Called by: resolve(), expireUnresolved()
// Effect: marks settled, records settledPayout, zeroes currentLiability

function claimWinnings(address winner, uint256 amount) external;
// Called by: claim()
// Effect: safeTransfer(winner, amount); revokes MARKET_ROLE when fully claimed

function asset() external view returns (address);
// Called by: initialize() — once, to cache the USDC address in the `usdc` slot.
//            usdcToken() view reads the cached slot directly; no live call is made.
// NOT called in any trading hot path.
```

**All four write functions are gated by `MARKET_ROLE` on the pool.** `MARKET_ROLE` is granted to the clone address during `pool.registerMarket()` and revoked automatically by the pool when the market is fully settled. If the pool revokes `MARKET_ROLE` prematurely (bug), all four write calls will revert.

---

*For theoretical foundation, see `BlieverMarket.md`. For math primitives, see `LSMath.sol`.*
