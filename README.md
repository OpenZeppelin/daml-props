# daml-props

Property-based testing for DAML. Pure DAML, no external dependencies.

Inspired by [Hedgehog](https://github.com/hedgehogqa/haskell-hedgehog) and [Echidna](https://github.com/crytic/echidna), daml-props generates random inputs, checks that your invariants hold for all of them, and shrinks failing cases to minimal counterexamples.

## Why Property-Based Testing?

Hand-written DAML Script tests validate specific inputs chosen by the developer. If you test a transfer with amounts 50.0 and 100.0, you haven't tested 0.0, 0.0000000001, or 9999999999.9999999999. Property-based testing generates hundreds of random inputs and asserts that invariants hold for every one. When a test fails, daml-props uses delta-debugging to shrink the counterexample to the smallest sequence that still fails.

## Installation

Clone and build:

```bash
git clone https://github.com/OpenZeppelin/daml-props.git
cd daml-props
dpm build
```

Add the built `.dar` as a dependency in your project's `daml.yaml`:

```yaml
dependencies:
  - daml-prim
  - daml-stdlib
  - path/to/daml-props/.daml/dist/daml-props-0.1.0.dar
```

Requires DAML SDK 3.4.10 (target 2.1). Build with `dpm build`, test with `dpm test`.

## Quick Start

### Simple Property

```daml
module MyProject.Test.Properties where

import DamlProps.Gen
import DamlProps.Property

-- For all integers in [1, 1000], doubling produces a larger number
test_doubleIsLarger = script do
  pure (assertProperty "double is larger" (genInt 1 1000) (\n -> n * 2 > n))
```

### Composing Generators

```daml
import DamlProps.Gen
import DamlProps.Property

-- For all pairs of positive decimals, addition is commutative
test_addCommutative = script do
  let gen = genPair (genDecimal 0.01 1000.0) (genDecimal 0.01 1000.0)
  pure (assertProperty "addition commutative" gen (\(a, b) -> a + b == b + a))
```

### Inspecting Results

```daml
import DamlProps.Gen
import DamlProps.Property

test_checkResult = script do
  let result = forAll "positive stays positive" (genDecimal 0.01 100.0) (\d -> d > 0.0)
  case result of
    PropSuccess count -> debug ("Passed " <> show count <> " tests")
    PropFailure _ seed msg -> error msg
```

## Sequence Testing

The core value for DAML: test random sequences of state transitions and check invariants after each step.

```daml
module MyProject.Test.SequenceProperties where

import DamlProps.Gen
import DamlProps.Sequence
import DamlProps.Runner

-- 1. Define domain actions
data BankAction
  = Deposit Decimal
  | Withdraw Decimal
  deriving (Show, Eq)

-- 2. Generator for random actions
genBankAction : Gen BankAction
genBankAction = genOneOf
  [ genMap Deposit (genDecimal 0.01 1000.0)
  , genMap Withdraw (genDecimal 0.01 500.0)
  ]

-- 3. State executor: apply action to state, return new state or error
execute : Decimal -> BankAction -> Either Text Decimal
execute balance (Deposit amt) = Right (balance + amt)
execute balance (Withdraw amt)
  | amt > balance = Left "Insufficient funds"
  | otherwise = Right (balance - amt)

-- 4. Invariant: balance is never negative
checkBalance : Decimal -> Either Text ()
checkBalance bal
  | bal < 0.0 = Left ("Negative balance: " <> show bal)
  | otherwise = Right ()

-- 5. Run 100 random sequences of up to 10 actions each
test_bankInvariant = script do
  pure (quickCheck "bank balance" genBankAction execute checkBalance 0.0)
```

When a sequence fails, the runner reports the seed, shrinks to the minimal failing sequence, and shows which invariant was violated.

## API Reference

### DamlProps.Gen

Generator combinators. No typeclasses needed — all plain functions.

| Function | Type | Description |
|----------|------|-------------|
| `genInt` | `Int -> Int -> Gen Int` | Int in [lo, hi] |
| `genBool` | `Gen Bool` | Random Bool |
| `genDecimal` | `Decimal -> Decimal -> Gen Decimal` | Decimal in [lo, hi] |
| `genAmount` | `Gen Decimal` | Positive amount (0.0000000001 to 10M) |
| `genSmallAmount` | `Gen Decimal` | Small positive (0.0000000001 to 1.0) |
| `genElement` | `[a] -> Gen a` | Random element from list |
| `genList` | `Int -> Int -> Gen a -> Gen [a]` | List of length in [lo, hi] |
| `genNonEmpty` | `Int -> Gen a -> Gen [a]` | Non-empty list, length in [1, hi] |
| `genMaybe` | `Int -> Gen a -> Gen (Optional a)` | Optional with None probability (0-100) |
| `genOneOf` | `[Gen a] -> Gen a` | Uniform choice among generators |
| `genFrequency` | `[(Int, Gen a)] -> Gen a` | Weighted choice |
| `genFilter` | `Int -> (a -> Bool) -> Gen a -> Gen a` | Filter with max retries |
| `genPair` | `Gen a -> Gen b -> Gen (a, b)` | Pair of generators |
| `genMap` | `(a -> b) -> Gen a -> Gen b` | Map over generator |
| `genBind` | `Gen a -> (a -> Gen b) -> Gen b` | Sequence generators |
| `genPure` | `a -> Gen a` | Lift pure value |

### DamlProps.Property

Pure property checking (no Script effects).

| Function | Type | Description |
|----------|------|-------------|
| `forAll` | `Text -> Gen a -> (a -> Bool) -> PropResult` | Run with defaults (100 tests) |
| `forAllWith` | `PropConfig -> Text -> Gen a -> (a -> Bool) -> PropResult` | Run with custom config |
| `assertProperty` | `Text -> Gen a -> (a -> Bool) -> ()` | Assert, calls `error` on failure |
| `assertPropertyShow` | `(a -> Text) -> Text -> Gen a -> (a -> Bool) -> ()` | Assert with counterexample display |
| `isSuccess` | `PropResult -> Bool` | Check if result is a pass |
| `isFailure` | `PropResult -> Bool` | Check if result is a failure |

### DamlProps.Runner

Top-level runner for stateful sequence testing.

| Function | Type | Description |
|----------|------|-------------|
| `quickCheck` | `Text -> Gen a -> (s -> a -> Either Text s) -> (s -> Either Text ()) -> s -> ()` | Quick check with defaults |
| `assertProp` | `RunnerConfig -> Gen a -> ... -> ()` | Assert with custom config |
| `runProp` | `RunnerConfig -> Gen a -> ... -> RunResult s a` | Run and return result |
| `formatResult` | `RunResult s a -> Text` | Format result for display |
| `formatSummary` | `RunResult s a -> Text` | Single-line summary |
| `isPassed` | `RunResult s a -> Bool` | Check if passed |
| `isFailed` | `RunResult s a -> Bool` | Check if failed |

### DamlProps.Sequence

Generic step type and sequence execution.

| Function | Type | Description |
|----------|------|-------------|
| `genSequence` | `Int -> Gen a -> Gen [Step a]` | Generate sequence up to maxLen |
| `runSequence` | `(s -> a -> Either Text s) -> (s -> Either Text ()) -> s -> [Step a] -> Either (FailureInfo s a) s` | Execute with invariant checking |
| `sequenceFails` | `... -> Bool` | Check if sequence triggers failure |
| `stepPayloads` | `[Step a] -> [a]` | Extract payloads from steps |

### DamlProps.Invariant

Built-in invariants and combinators.

| Function | Type | Description |
|----------|------|-------------|
| `conservationInvariant` | `TransferInvariant` | sum(inputs) == sum(outputs) |
| `positiveAmountInvariant` | `Invariant` | All holdings > 0 |
| `nonEmptyInputInvariant` | `TransferInvariant` | Transfer has >= 1 input |
| `nonNegativeBalanceInvariant` | `Invariant` | No party has negative balance |
| `supplyConservationInvariant` | `Decimal -> Invariant` | Total supply unchanged |
| `combineInvariants` | `[Invariant] -> Invariant` | Combine, fail on first violation |
| `labelInvariant` | `Text -> Invariant -> Invariant` | Label for error messages |

### DamlProps.Shrink

Delta-debugging shrinker and value shrink candidates.

| Function | Type | Description |
|----------|------|-------------|
| `shrinkSequence` | `ShrinkConfig -> ... -> [Step a]` | Minimize failing sequence |
| `shrinkValue` | `(a -> Bool) -> [a] -> Optional a` | Find smallest failing candidate |
| `shrinkInt` | `Int -> [Int]` | Shrink candidates toward 0 |
| `shrinkDecimal` | `Decimal -> [Decimal]` | Shrink candidates toward 0.0 |
| `shrinkList` | `[a] -> [[a]]` | Shrink by removing elements |

### DamlProps.Seed

PRNG wrapper (typically not used directly).

| Function | Type | Description |
|----------|------|-------------|
| `mkSeed` | `Int -> Seed` | Create seed from Int |
| `nextInt` | `Seed -> (Int, Seed)` | Next random Int |
| `nextInRange` | `Int -> Int -> Seed -> (Int, Seed)` | Int in [lo, hi] |
| `split` | `Seed -> (Seed, Seed)` | Split for independent streams |
| `nextDecimal` | `Seed -> (Decimal, Seed)` | Decimal in [0, 1) |
| `nextDecimalInRange` | `Decimal -> Decimal -> Seed -> (Decimal, Seed)` | Decimal in [lo, hi) |

## Dogfooding

daml-props has been validated against two production codebases:

### canton-network-token-standard (simple-token)

Pure state-machine model of the CIP-056 transfer engine. 4 parties, 6 action types (self-transfer, direct transfer, two-step initiate/accept/reject/withdraw). 5 property tests, all passing. See `Examples/`.

### Splice (amulet)

Pure model of `AmuletRules.daml` transfer engine with buggy and fixed executor variants. daml-props detected the H2 vulnerability through property-based testing: zero-input featured-app transfers mint reward coupons without burning any amulet. The `test_amuletH2Buggy` negative test catches this with a minimal 1-step sequence. See [SPLICE_FINDINGS.md](SPLICE_FINDINGS.md) and `DamlProps/Examples/Amulet/`.

### Integration Guide

To use daml-props in your own test package:

1. Build daml-props: `cd daml-props && dpm build`
2. Add the DAR to your `daml.yaml`:
   ```yaml
   data-dependencies:
     - path/to/daml-props/.daml/dist/daml-props-0.1.0.dar
   ```
3. Write a pure state-machine model (executor returns `Right state` for invalid preconditions, `Left` for bugs)
4. Write property tests using `runProp config generator executor invariant initialState`
5. Run: `dpm build && dpm test`

## Requirements

- DAML SDK 3.4.10 (target: 2.1)
- Build tool: [`dpm`](https://github.com/OpenZeppelin/dpm) (not the deprecated `daml` CLI)
- Dependencies: `daml-prim`, `daml-stdlib`, `daml-script`

## License

Apache-2.0. See [LICENSE](LICENSE) for details.
