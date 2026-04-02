# [H-1] Delegators Can Never Withdraw Their Funds Due to Incorrect Delegation Mapping

## Summary
A logic bug in the delegation process makes it impossible for delegators to ever withdraw their staked funds. Regardless of validator behavior, delegators are permanently unable to reclaim their stake.


## Finding Description
The `ConsensusRegistry` contract implements a delegation mechanism where a delegator can delegate stake to a validator. The intended behavior is that the delegator should retain control over their staked funds and be able to withdraw them, or at least receive them back if the validator or delegator calls `unstake`.

However, in the `delegateStake` function, the assignment to the `Delegation` struct is incorrect:

```solidity
delegations[validatorAddress] = Delegation(blsPubkeyHash, msg.sender, validatorAddress, validatorVersion, nonce + 1);
```

Here, `msg.sender` (the delegator) is assigned to the `validatorAddress` field, and `validatorAddress` (the validator) is assigned to the `delegator` field. This is the reverse of the intended mapping.

As a result, when the contract later checks for the recipient in `_getRecipient`:

```solidity
function _getRecipient(address validatorAddress) internal view returns (address) {
    Delegation storage delegation = delegations[validatorAddress];
    address recipient = delegation.delegator;
    if (recipient == address(0x0)) recipient = validatorAddress;
    return recipient;
}
```

the recipient is always the validator, not the delegator. This means that when `unstake` is called, the funds are always sent to the validator, even if the stake was provided by a delegator.


The delegator is simply unable to ever reclaim their funds, as the contract logic will always send the funds to the validator, and the permission check will never allow the delegator to withdraw.

Also on the `ConsensusRegistry` contract, when calling the claimStakeRewards, in case of a delegated stake, the delegator won't get any rewards, since the recipient is the validator, the validator will get all rewards and staking balance after unstaking. 

```solidity
// require caller is either the validator or its delegator
address recipient = _getRecipient(validatorAddress);
if (msg.sender != validatorAddress && msg.sender != recipient) revert NotRecipient(recipient);
```

This breaks the security guarantee that delegated funds remain under the control of the delegator. A delegator can never recover their funds, even if they try to call `unstake` themselves.

## Impact Explanation
This is a **High** vulnerability because it allows validators to steal all funds delegated to them, and delegators lose all control over their stake. Even without validator malice, delegators are permanently unable to recover their funds, violating the core trust and security model of staking and delegation.

## Likelihood Explanation
The likelihood is **High** because:
- The bug is present in the main delegation flow.
- Delegators have no way to recover their funds once delegated, even if they try to withdraw themselves.

## Proof of Concept

Add the following test function to the  ConsensusRegistryTest.t.sol contract on the root `test/consensus/ConsensusRegistry.t.sol`

```solidity
    function test_delegatorsCanNeverCaimTheirFunds() public {
        vm.prank(crOwner);
        uint256 validator5PrivateKey = 5;
        validator5 = vm.addr(validator5PrivateKey);
        address delegator = _addressFromSeed(42);
        vm.deal(delegator, stakeAmount_);
        assert(delegator.balance == stakeAmount_);
        consensusRegistry.mint(validator5);

        bytes32 structHash = consensusRegistry.delegationDigest(validator5BlsPubkey, validator5, delegator);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(validator5PrivateKey, structHash);
        bytes memory validatorSig = abi.encodePacked(r, s, v);

        // Check event emission
        bool isDelegate = true;
        vm.expectEmit(true, true, true, true);
        emit ValidatorStaked(
            ValidatorInfo(
                validator5BlsPubkey,
                validator5,
                PENDING_EPOCH,
                uint32(0),
                ValidatorStatus.Staked,
                false,
                isDelegate,
                uint8(0)
            )
        );
        vm.prank(delegator);
        consensusRegistry.delegateStake{ value: stakeAmount_ }(validator5BlsPubkey, validator5, validatorSig);
        assert(delegator.balance == 0);

        ValidatorInfo[] memory validators = consensusRegistry.getValidators(ValidatorStatus.Staked);
        
        //Test delegator cannot withdraw their funds
        vm.prank(delegator);
        vm.expectRevert(abi.encodeWithSelector(IStakeManager.NotRecipient.selector, validator5));
        consensusRegistry.unstake(validator5);

        //Test vaidator can unstake, and recieve funds (because he is the reciever, but the delegator should be)
        vm.prank(validator5);
        consensusRegistry.unstake(validator5);

        assert(delegator.balance == 0);
        assert(validator5.balance == stakeAmount_);

    }
```

## Recommendation

**Fix the assignment in `delegateStake` to correctly map the validator and delegator:**

```solidity
// Correct assignment
delegations[validatorAddress] = Delegation(blsPubkeyHash, validatorAddress, msg.sender, validatorVersion, nonce + 1);
```


# [H-2] Double Penalty on Slashed Validators Prevents Reward Withdrawal

## Summary
The staking reward calculation does not account for slashed amounts, causing slashed validators to be unable to claim any rewards until their balance exceeds their initial stake. This results in an unintended double penalty.

## Finding Description
The vulnerability is in the reward calculation logic for validators. When a validator is slashed, their `balance` is reduced, but the calculation for claimable rewards still uses the original staked amount as a baseline. This means that any future rewards are first used to “fill” the slashed gap, and only after the balance surpasses the initial stake can the validator claim rewards.

This breaks the expected security guarantee that slashing should only penalize the validator by the slashed amount. Instead, it also blocks access to future rewards until the slash is “repaid,” effectively penalizing the validator twice. The root cause is at line 227 of `StakeManager.sol`, where the calculation is:

```solidity
 uint256 rewards = balance > initialStake ? balance - initialStake : 0;
```

## Impact Explanation
**Impact: High**

Slashed validators are penalized more than intended, as they lose both the slashed amount and access to their earned rewards until their balance exceeds the initial stake. This can discourage participation and trust in the protocol, and may be considered a protocol-level economic bug.

## Likelihood Explanation
**Likelihood: High**

Slashing is a core part of staking protocols, and it is expected that validators will be slashed from time to time. Any validator who is slashed and then continues to participate and earn rewards will encounter this issue. The bug is deterministic and will affect all slashed validators, making it highly likely to occur in practice.

## Proof of Concept

Add the following test function to the ConsensusRegistryTest.t.sol contract on the root `test/consensus/ConsensusRegistry.t.sol`

```solidity
    function testStakerSlashedHaveDoublePenalisationWhenClaimingRewards() external {
        vm.prank(crOwner);
        consensusRegistry.mint(validator5);

        assertEq(consensusRegistry.getValidators(ValidatorStatus.Staked).length, 0);

        // Check event emission
        vm.expectEmit(true, true, true, true);
        emit ValidatorStaked(
            ValidatorInfo(
                validator5BlsPubkey,
                validator5,
                PENDING_EPOCH,
                uint32(0),
                ValidatorStatus.Staked,
                false,
                false,
                uint8(0)
            )
        );
        vm.prank(validator5);
        consensusRegistry.stake{ value: stakeAmount_ }(validator5BlsPubkey);

        Slash[] memory slashes = new Slash[](1);

        slashes[0] = Slash({
            validatorAddress: validator5,
            amount: 200_000e18
        });

        RewardInfo[] memory incentives = new RewardInfo[](1);

        incentives[0] = RewardInfo({
            validatorAddress: validator5,
            consensusHeaderCount: 5
        });
        
        vm.startPrank(0xffffFFFfFFffffffffffffffFfFFFfffFFFfFFfE, 0xffffFFFfFFffffffffffffffFfFFFfffFFFfFFfE);  // -> System address
        consensusRegistry.applySlashes(slashes);
        assert(consensusRegistry.getBalance(validator5) == 1_000_000e18 - 200_000e18);

        //User get slashed for 200_000e18 (can be any other amount, just to be more explicit i used big numbers)
        consensusRegistry.applyIncentives(incentives);
        vm.stopPrank();

        assert(consensusRegistry.getBalance(validator5) == 1_000_000e18 - 200_000e18 + 25_806e18);
        //At this point rewards should be 25_806e18

        assert(consensusRegistry.getRewards(validator5) == 0);
        //This returns zero because is calculating rewards based on the initial staked balanece, withouth taking into consideration the slashed amount
        vm.prank(validator5);
        vm.expectRevert(abi.encodeWithSelector(IStakeManager.InsufficientRewards.selector, 0));
        consensusRegistry.claimStakeRewards(validator5);

        //The function that calculate rewards return 0 because balance is lower than the initialStakd amount and reverts


    }
```

## Recommendation
Update the reward calculation logic to account for slashed amounts. One possible fix is to track the total amount slashed for each validator and subtract it from the initial stake when calculating claimable rewards. For example:

```solidity
// Pseudocode for fix
uint256 effectiveStake = initialStake - totalSlashed[validatorAddress];
uint256 rewards = balances[validatorAddress] > effectiveStake
    ? balances[validatorAddress] - effectiveStake
    : 0;
```

This ensures that slashing only penalizes the validator by the intended amount and does not block access to future rewards.


# [L-1] Committee Size Calculation Bug in `_ejectFromCommittees`

# Vulnerability Report: Committee Size Calculation Bug in `_ejectFromCommittees`

## Summary
The `_ejectFromCommittees` function contains a bug in the committee size calculation after ejecting a validator, which can result in incorrect committee size calculation and potentially cause issues in committee validation.

## Finding Description
In the `_ejectFromCommittees` function of the `ConsensusRegistry.sol` contract, lines 490-491, there is a logical error in the committee size calculation after ejecting a validator:

```solidity
bool ejected = _eject(currentCommittee, validatorAddress);
uint256 committeeSize = ejected ? currentCommittee.length - 1 : currentCommittee.length;
```

The problem is that the `_eject()` function directly modifies the storage array `currentCommittee` through `committee.pop()`, which automatically reduces `currentCommittee.length` by 1. By subtracting 1 additional when `ejected` is `true`, the code is subtracting the array size **twice**.

This bug breaks the security guarantee that committee sizes are consistent and valid. When `_checkCommitteeSize()` is called subsequently, it may pass with an incorrect committee size that is 1 less than the actual value, which can result in the acceptance of invalid committees.

## Impact Explanation
**Severity: Low**

## Likelihood Explanation
**Likelihood: Medium**

This bug is likely to occur in the following situations:
- When the `owner` executes the `burn()` function to eject a validator
- When the system executes `applySlashes()` to penalize malicious validators
- During maintenance operations of the consensus system

## Proof of Concept
Consider a scenario where a committee has 10 validators and one validator needs to be ejected. The current committee size is 10. When `_eject()` is called, it removes the validator from the array using `pop()`, which automatically reduces the array length to 9. However, the buggy code then subtracts 1 again from the already updated length, resulting in `committeeSize = 8` instead of the correct value of 9. This incorrect size is then passed to `_checkCommitteeSize()`, which may incorrectly validate the committee size and potentially allow the formation of invalid committees that don't match the actual number of available validators.

## Recommendation
The bug can be fixed in two ways:

**Option 1:** Use the updated size directly
```solidity
bool ejected = _eject(currentCommittee, validatorAddress);
uint256 committeeSize = currentCommittee.length; // Already updated by _eject()
```

**Option 2:** Save the original size before ejection
```solidity
uint256 originalSize = currentCommittee.length;
bool ejected = _eject(currentCommittee, validatorAddress);
uint256 committeeSize = ejected ? originalSize - 1 : originalSize;
```

Option 1 is simpler and more efficient, as `_eject()` already updates the storage array size.

## Related Files

- `tn-contracts/src/consensus/ConsensusRegistry.sol` (lines 490-490)


