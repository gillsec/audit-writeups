## Redundant MINIMUM_LIQUIDITY Check Enables Griefing via DoS

**Severity:** Medium (downgraded to Low by judge)
**Platform:** Bug Bounty — Immunefi
---

### Summary

The `mint` function in `Pool.sol` applies the `MINIMUM_LIQUIDITY` check globally to all callers, not just the initial depositor. An attacker can exploit this by initializing the pool with a small amount, then donating a large amount of tokens directly to the contract to artificially inflate the share price. This causes legitimate users' liquidity calculations to fall below the 1,000-unit threshold, reverting their transactions and effectively bricking deposits for a wide range of deposit sizes.

---

### Vulnerability Details

The issue lies in the following logic:
```solidity
if (_totalSupply == 0) {
    liquidity = Math.sqrt(_amount0 * _amount1) - MINIMUM_LIQUIDITY;
    _mint(address(1), MINIMUM_LIQUIDITY);
} else {
    liquidity = Math.min((_amount0 * _totalSupply) / _reserve0, (_amount1 * _totalSupply) / _reserve1);
}

if (liquidity < MINIMUM_LIQUIDITY) revert InsufficientLiquidityMinted(); // @audit applied to ALL callers
_mint(to, liquidity);
```

The `MINIMUM_LIQUIDITY` threshold is intended only for the initial mint to prevent inflation attacks. Applying it to every subsequent deposit creates a permanent manipulation vector.

**Attack Steps:**
1. Attacker initializes the pool with a minimal deposit
2. Attacker donates a large amount of tokens directly to the contract, bypassing `mint`
3. The inflated reserves cause subsequent liquidity calculations to return values below `MINIMUM_LIQUIDITY`
4. All legitimate deposits revert with `InsufficientLiquidityMinted`

---

### Impact

Griefing / Denial of Service on core liquidity provision. Legitimate users are unable to deposit funds into the pool. Attack cost is low — the attacker retains nearly all LP shares minus the burned `MINIMUM_LIQUIDITY` units, and can recover most capital by withdrawing after the donation.

---

### Severity Rationale

Submitted as **Medium**. Downgraded to **Low** by the project on the basis that executing the attack requires the attacker to lock tokens as minimum liquidity, incurring a loss.

I maintain the **Medium** classification given the disproportionate impact relative to attack cost and the absence of a complete mitigation at the contract level.