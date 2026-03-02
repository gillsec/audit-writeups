# Swapped Arguments in QuadraticSwapFees Enable K Invariant Violation

**Severity:** High  
**Platform:** Audit Contest — Cantina (Amalgam)

---

## Summary

The arguments `referenceReserve` and `reserve` are passed in the wrong order when calling `QuadraticSwapFees.calculateSwapFeeBipsQ64`, causing incorrect fee calculations that can allow swaps to violate the K invariant or unfairly revert legitimate ones.

---

## Vulnerability Details

The AMM relies on `QuadraticSwapFees.calculateSwapFeeBipsQ64` to compute swap fees, which are critical for maintaining the economic security of the pool. The function expects `(amount, referenceReserve, currentReserve)`, but the call site passes `currentReserve` and `referenceReserve` in swapped positions.

As a result, instead of calculating the fee based on the actual price movement relative to the start-of-block reference, the function uses an inverted reference point — producing fees that are either too low or too high depending on reserve conditions.

---

## Impact

When the fee is calculated too low, an attacker can perform swaps that violate the K invariant, extracting more value from the pool than is economically justified. When calculated too high, legitimate swaps that would maintain the K invariant are incorrectly reverted. In both cases, liquidity providers suffer — either through direct value extraction or reduced trading activity. The bug is particularly dangerous during high volatility or low liquidity periods, and may not always produce obvious symptoms, allowing the pool to leak value over time.

---

## Recommendation

Review every call to `QuadraticSwapFees.calculateSwapFeeBipsQ64` and ensure arguments are passed in the correct order:

```solidity
// Correct
QuadraticSwapFees.calculateSwapFeeBipsQ64(amount, referenceReserve, currentReserve);

// Incorrect (current implementation)
QuadraticSwapFees.calculateSwapFeeBipsQ64(amount, currentReserve, referenceReserve);
```