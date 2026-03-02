## SuperValidatorBase Incorrectly Handles Infinite Validity Signatures (validUntil=0) Against ERC7579 Specification

**Severity:** Medium
**Platform:** Audit Contest — Cantina (Superform V2)

---

### Summary

The `SuperMerkleValidator` contract incorrectly handles signatures with `validUntil = 0`, which per the ERC7579 specification should indicate infinite validity. Due to a flawed comparison in `_isSignatureValid()`, these signatures are always rejected since `0 >= block.timestamp` is always false.

---

### Vulnerability Details

The `_isSignatureValid()` function compares `validUntil` directly against `block.timestamp`:
```solidity
return signer == _accountOwners[sender] && validUntil >= block.timestamp;
```

When `validUntil = 0`, this condition always evaluates to false, causing all infinite-validity signatures to be rejected. This directly violates the ERC7579ValidatorBase specification, which explicitly documents `0` as meaning "no expiration."

---

### Impact

Any signature intended to have infinite validity is permanently rejected. This breaks core validation functionality, forces users to set arbitrary expiration timestamps, and can cause denial of service for operations that require non-expiring authorizations.

---

### Recommendation
```solidity
return signer == _accountOwners[sender] && (validUntil == 0 || validUntil >= block.timestamp);
```