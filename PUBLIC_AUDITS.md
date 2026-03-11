# daml-props — Public Audit Results

Property-based testing of DAML contracts using random action sequences, invariant checking, and delta-debugging shrinkage. All findings are **NOT VALIDATED** by the respective project teams — they are tool output only.

**Date:** 2026-03-04
**Tool version:** daml-props v0.1.0 (pure DAML, SDK 3.4.10)
**Methodology:** Generate random action sequences, check invariants after each step, shrink failures to minimal reproductions.

---

## Table of Contents

1. [Validated Findings: Splice](#1-validated-findings-splice)
2. [Validated Findings: canton-network-token-standard](#2-validated-findings-canton-network-token-standard)
3. [Coverage Gaps and Recommendations](#3-coverage-gaps-and-recommendations)
4. [Recommended Property Tests: canton-dex-platform](#4-recommended-property-tests-canton-dex-platform)
5. [Recommended Property Tests: canton-erc20](#5-recommended-property-tests-canton-erc20)
6. [Recommended Property Tests: canton-vault](#6-recommended-property-tests-canton-vault)
7. [Recommended Property Tests: daml-finance](#7-recommended-property-tests-daml-finance)
8. [Recommended Property Tests: splice (Extensions)](#8-recommended-property-tests-splice-extensions)

---

## 1. Validated Findings: Splice

**GitHub:** [hyperledger-labs/splice](https://github.com/hyperledger-labs/splice)
**Commit:** [`6a0dcbba4`](https://github.com/hyperledger-labs/splice/commit/6a0dcbba4bdae0861cd6f5021bfd436c18fd5034)
**Target:** AmuletRules transfer engine (`checkTransferConstraints` + reward minting)

### Summary

| Test | Property | Model | Result |
|------|----------|-------|--------|
| `test_amuletH2Buggy` | No rewards without burn | Buggy (mirrors splice) | **FAIL (expected)** |
| `test_amuletH2Fixed` | No rewards without burn | Fixed (adds min input check) | PASS |
| `test_amuletFeeNonNegativity` | Total burned fees >= 0 | Fixed | PASS |
| `test_amuletMaxConstraints` | Input/output counts within config limits | Fixed | PASS |

### H2 Vulnerability: Zero-Input Transfer Mints Rewards

**The vulnerability:** `checkTransferConstraints` in `AmuletRules.daml` (lines 783-788) validates that input/output counts don't exceed maximums, but does **not** check for a minimum of 1 input. A featured-app transfer with zero inputs and zero outputs:

1. Passes all constraint checks (0 <= maxInputs, 0 <= maxOutputs)
2. Costs zero fees (no outputs to charge)
3. Mints `extraAppReward` (16.5 amulet) as a reward coupon
4. Burns nothing

**Result:** Free reward minting from thin air.

### Detection Methodology

Two executor variants were created:

**Buggy executor** (mirrors splice):
```
executeAmuletBuggy state action =
  if length inputs > maxNumInputs then Right state  -- no-op
  else if length outputs > maxNumOutputs then Right state  -- no-op
  else executeAmuletCore state action  -- zero inputs ALLOWED
```

**Fixed executor** (adds minimum input check):
```
executeAmuletFixed state action =
  if null inputs then Right state  -- reject zero inputs
  else if length inputs > maxNumInputs then Right state
  else if length outputs > maxNumOutputs then Right state
  else executeAmuletCore state action
```

**Invariant:** `noRewardsWithoutBurn` — if `totalMinted > 0` but `totalBurned == 0`, rewards were minted for free.

### Minimal Failing Sequence

1 step after delta-debugging shrinkage:

```
AmuletAction { atSender = 1, atInputAmounts = [], atOutputs = [], atIsFeaturedApp = True }
```

The generator `genZeroInputTransfer` produces this action with 30% probability (weighted 3/10 via `genFrequency`). daml-props shrinks longer failing sequences down to this minimal 1-step reproduction.

### Model Details

**State:**
```
AmuletState:
  amulets: [(partyIdx, amount)]     -- active holdings
  rewardCoupons: [Decimal]          -- minted reward amounts
  totalBurned: Decimal              -- cumulative fees burned
  totalMinted: Decimal              -- cumulative rewards minted
  config: AmuletConfig              -- fee rates + limits
```

**Configuration:**
```
AmuletConfig:
  createFee = 0.03                  -- per-output creation fee
  transferFeeRate = 0.001           -- percentage fee on output amounts
  extraAppReward = 16.5             -- reward for featured app transfers
  maxNumInputs = 100                -- maximum inputs per transfer
  maxNumOutputs = 100               -- maximum outputs per transfer
```

**Generators:**

| Generator | Weight | Description |
|-----------|--------|-------------|
| `genNormalTransfer` | 4/10 | 1-3 inputs, 1 output, non-featured |
| `genZeroInputTransfer` | 3/10 | 0 inputs, 0 outputs, featured (H2 trigger) |
| `genFeaturedTransfer` | 2/10 | 1-2 inputs, 1 output, featured |
| `genLargeTransfer` | 1/10 | 5-20 inputs, 1-5 outputs, non-featured |

### Recommended Fix

Add `require "at least one input" (not $ null inputs)` check in `checkTransferConstraints`.

### Limitations

- **Pure model only.** Tests a state-machine model, not actual DAML contracts on a ledger. Cannot catch authorization or time-dependent issues.
- **Simplified fee model.** Models `createFee + transferFeeRate` but not stepped rates or synchronizer fees.
- **No governance interaction.** Does not model DSO configuration changes or round advancement.

---

## 2. Validated Findings: canton-network-token-standard

**GitHub:** [OpenZeppelin/canton-network-token-standard](https://github.com/OpenZeppelin/canton-network-token-standard)
**Target:** CIP-056 simple-token transfer engine

Pure state-machine model of the CIP-056 transfer engine. 4 parties, 6 action types (self-transfer, direct transfer, two-step initiate/accept/reject/withdraw). 5 property tests, all passing.

| Test | Property | Result |
|------|----------|--------|
| `test_conservation` | sum(inputs) == sum(outputs) | PASS |
| `test_positiveAmount` | All holdings > 0 | PASS |
| `test_nonEmptyInput` | Transfer has >= 1 input | PASS |
| `test_nonNegativeBalance` | No party has negative balance | PASS |
| `test_supplyConservation` | Total supply unchanged | PASS |

See the repository for full details.

---

## 3. Coverage Gaps and Recommendations

| Project | daml-props Status | Priority |
|---------|------------------|----------|
| canton-dex-platform | No tests | Medium |
| canton-erc20 | No tests | High — CIP-056 compliance gaps identified by daml-lint |
| canton-vault | No tests | High — division-by-zero risk in share calculations |
| daml-finance | No tests | Low — library with broad parameter space |
| splice | H2 tested | Medium — extend to governance config changes |

---

## 4. Recommended Property Tests: canton-dex-platform

**GitHub:** [0xalberto/canton-dex-platform](https://github.com/0xalberto/canton-dex-platform)
**Commit:** [`e9f01be`](https://github.com/0xalberto/canton-dex-platform/commit/e9f01be60369b9978dcf0e60aa5fa2ce02967a3f)

### Model: Account Balance State Machine

**State:** `Map Party Decimal` (account balances)
**Actions:** `Reserve`, `Release`, `Transfer`

**Recommended properties:**
- **Conservation:** `TransferFunds` preserves total balance across sender + recipient: `balanceBefore(sender) + balanceBefore(recipient) == balanceAfter(sender) + balanceAfter(recipient)`
- **Positive balance:** Account balance never goes negative after any sequence of Reserve/Release/Transfer
- **Settlement atomicity:** DvP settlement either completes fully or fails fully — no partial settlements

### Model: Settlement Lifecycle

**State:** `(Map SettlementId SettlementStatus, Map AccountId Decimal)`
**Actions:** `CreateOrder`, `MatchOrder`, `CreateSettlement`, `ExecuteDvP`, `FailSettlement`

**Recommended property:** No settlement remains in pending status indefinitely (liveness).

---

## 5. Recommended Property Tests: canton-erc20

**GitHub:** [ChainSafe/canton-erc20](https://github.com/ChainSafe/canton-erc20)
**Commit:** [`e98a7db`](https://github.com/ChainSafe/canton-erc20/commit/e98a7db5b9e0bdb96331cac72052a39134a84dfc)

### Model: Mint/Burn Conservation

**State:** `(totalMinted : Decimal, totalBurned : Decimal, holdings : Map Party Decimal)`
**Actions:** `Mint`, `Transfer`, `Burn`, `Deposit`, `Withdrawal`

**Recommended properties:**
- **Conservation:** `totalMinted == sum(holdings) + totalBurned` after any sequence
- **Bridge consistency:** Every `PendingDeposit` results in exactly one `DepositReceipt` and one `CIP56Holding`
- **No orphaned events:** Every `WithdrawalEvent` has a corresponding token burn
- **Positive holdings:** All CIP-056 holdings > 0 (compensates for missing `ensure` clause)

---

## 6. Recommended Property Tests: canton-vault

**GitHub:** [ted-gc/canton-vault](https://github.com/ted-gc/canton-vault)
**Commit:** [`bb51d2c`](https://github.com/ted-gc/canton-vault/commit/bb51d2c5df89d506dc38af8a3760029013c1ffe7)

### Model: Deposit/Redeem Share Conservation

**State:** `VaultState { totalAssets : Decimal, totalShares : Decimal }`
**Actions:** `Deposit`, `Mint`, `Redeem`, `Withdraw`, `AccrueFees`

**Recommended properties:**
- **Share conservation:** `totalShares * sharePrice ≈ totalAssets` (within rounding tolerance) after any deposit/redeem sequence
- **No free shares:** Zero-amount deposits produce zero shares
- **Withdrawal completeness:** Redeeming all shares returns all deposited assets (minus fees)
- **No empty vault with shares:** `totalAssets == 0 → totalShares == 0` (catches the division-by-zero risk)
- **Rounding favor:** `calculateShares` always rounds down (vault never loses value to rounding)

---

## 7. Recommended Property Tests: daml-finance

**GitHub:** [digital-asset/daml-finance](https://github.com/digital-asset/daml-finance)
**Commit:** [`155f931b`](https://github.com/digital-asset/daml-finance/commit/155f931b6ebe7d3662fd72788cb17f0bfb5a7ba6)

### Model: Settlement Instruction Execution

**State:** `Map InstructionId InstructionStatus`
**Actions:** `CreateBatch`, `Allocate`, `Approve`, `Execute`, `Cancel`

**Recommended properties:**
- **Batch atomicity:** All instructions in a batch settle or none do
- **No double-allocation:** An instruction cannot be allocated twice
- **Settlement completeness:** All approved instructions eventually execute or cancel

**Note:** daml-finance has an extremely broad parameter space (bonds, options, swaps, structured products). Property tests should focus on the settlement pipeline rather than individual instrument lifecycle.

---

## 8. Recommended Property Tests: splice (Extensions)

Beyond the validated H2 tests, recommended extensions:

### Governance Configuration Changes

**State:** `AmuletConfig` + `VaultState`
**Actions:** `ProposeConfigChange`, `VoteOnConfig`, `ExecuteConfigChange`, `Transfer`

**Recommended properties:**
- **No zero-price:** `amuletPrice` never reaches 0.0 after any governance sequence
- **Fee monotonicity:** Fee changes don't retroactively affect in-flight transfers
- **Issuance cap safety:** `capPerCoupon > 0.0` maintained after any config change sequence

### Reward System

**Actions:** `Transfer`, `MintReward`, `ClaimReward`, `ExpireReward`

**Recommended properties:**
- **No rewards without activity:** Reward coupons only minted when actual transfer activity occurs
- **Reward conservation:** Total claimed rewards <= total issued rewards
- **Expiry completeness:** All expired reward coupons eventually cleaned up

---

*Generated by daml-props v0.1.0 | Audit date: 2026-03-04*
