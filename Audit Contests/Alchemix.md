# [H-1] Unbounded Minting of alETH Due to Missing Access Control on `alEth:setWhitelist`

## Description

## Summary

Anyone can call the function setWhitelist, beeing able to mint any amount of alEth token.

## Finding Description

The `AlEth` contract contains a critical access control vulnerability in the `setWhitelist` function. This function is publicly accessible and lacks any `role-based` restrictions or modifiers to limit its use to authorized addresses (e.g., ADMIN). As a result, any address can call setWhitelist and add itself (or any other address) to the whitelist.

Once whitelisted, a user can call the `mint` function, which also lacks ceiling validation, allowing unlimited minting of the `alETH` token without any economic cost, collateral, or governance approval.

This breaks the token's intended monetary policy and opens the door to protocol-wide inflation, draining of liquidity pools, manipulation of yield-bearing systems, and potentially causing severe downstream effects across DeFi integrations.

Additionally, the setCeiling, pauseAlchemist, and similar administrative functions also lack proper access control, which could further escalate the attack surface.

-------------------------------------------------------------------------------------------

Anyone can call this function beeing able to ad itself or other address to the whitelist

```javascript
    function setWhitelist(address _toWhitelist, bool _state) external {
        whiteList[_toWhitelist] = _state;
    }
```
Once whitelisted, the attacker can mint any amount of alMint 

```javascript
    function mint(address _recipient, uint256 _amount) external //@> onlyWhitelisted {
        require(!paused[msg.sender], "AlETH: Alchemist is currently paused.");
        hasMinted[msg.sender] = hasMinted[msg.sender] + _amount;
        _mint(_recipient, _amount);
    }
```

## Impact Explanation

This vulnerability allows any external address to whitelist itself and mint an unlimited amount of `alETH` tokens without any form of economic backing or governance approval.

The consequences of this issue are severe:

- **Unlimited Inflation**: The attacker can arbitrarily inflate the token supply, undermining the token’s economic design and monetary policy.
- **Market Manipulation**: The ability to mint at will can be used to manipulate the price of `alETH`, affecting price pegged
- **Collateral Bypass**: The minting bypasses all collateral mechanisms, allowing attackers to extract value without locking or repaying any assets.
- **Systemic Risk**: Any integrations, lending markets, or protocols that rely on the integrity of `alETH` may be exposed to loss or insolvency.

The attacker gains full control over the issuance of the protocol’s core asset, which constitutes a critical severity vulnerability.


## Likelihood Explanation

The likelihood of exploitation is **high**, given that:

- The `setWhitelist` function is publicly accessible and lacks access control, allowing **any external address** to whitelist itself or others.
- The `mint` function only checks that the caller is whitelisted, without enforcing any ceiling or authorization from a privileged role.
- There are no time delays, governance checks, or external dependencies that would hinder an attacker from executing the exploit immediately after deployment.
- The exploit requires **no upfront capital**, making it highly attractive to malicious actors.

If deployed in production or in any environment with liquidity, the vulnerability could be exploited in a single transaction and result in irreversible economic damage.

## Proof of Concept


An attacker can exploit this vulnerability by following these simple steps:

1. **Call `setWhitelist(address attacker, true)`**  
   Since the function lacks any access control, the attacker can whitelist themselves as an approved minter.

2. **Call `mint(address attacker, arbitraryAmount)`**  
   Once whitelisted, the attacker can call the `mint` function to mint any desired amount of `alETH` tokens to their own address.

3. **Repeat or automate the process**  
   The attacker can continue to mint tokens without restriction, as the contract does not enforce any ceiling, rate limit, or governance approval.

4. **Dump tokens into the market**  
   The attacker can then exchange the illegitimately minted tokens for ETH, stablecoins, or other assets in available liquidity pools, draining protocol or community funds.

This process does not require any privileged role, collateral, or complex conditions. It can be performed entirely off-chain using basic scripts or manually via a block explorer interface like Etherscan or Tenderly. The exploit could be executed within a single block.

## Proof of Code

Create a new file `.t.sol` in the `/src/test` folder and paste the following code

```javascript
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.26;

import {Test} from "forge-std/Test.sol";
import {AlEth} from "src/external/AlEth.sol";

contract Poc_1 is Test {
    AlEth alEth;
    address attacker = makeAddr("attacker");
    function setUp() external {
        alEth = new AlEth();
    }

    function test_anyoneCanMintUndefinedAmountOfAleth() external {
        assert(alEth.balanceOf(attacker) == 0);

        vm.startPrank(attacker);
        alEth.setWhitelist(attacker, true);
        alEth.mint(attacker, type(uint256).max);
        vm.stopPrank();

        assert(alEth.balanceOf(attacker) == type(uint256).max);
    }
}
```


## Recommendation

To mitigate this vulnerability and prevent unauthorized minting, implement strict access control using role-based permissions. Specifically:

1. **Restrict `setWhitelist` to an admin-only role**  
   This function should only be callable by an account with explicit administrative privileges, such as `ADMIN_ROLE`.

2. **Validate minting limits in the `mint` function**  
   Enforce a ceiling by checking that `hasMinted[msg.sender] + _amount <= ceiling[msg.sender]` before allowing any mint.

3. **Implement role-based access using OpenZeppelin’s `AccessControl` or `Ownable`**  
   Define clear roles (e.g., `MINTER_ROLE`, `ADMIN_ROLE`) and restrict sensitive functions accordingly.

By implementing these controls, the protocol will be protected from unauthorized issuance of `alETH` and the integrity of its monetary system will be preserved.

# [H-2] Unauthorized Collateral Deduction in AlchemistV3

## Description

## Summary

The `repay()` function in AlchemistV3.sol contains a critical design flaw that causes protocol fees to be improperly deducted from user collateral instead of the approved yield tokens, resulting in silent loss of user funds and accounting inconsistencies.

## Finding Description

The `ÀlchemistV3:repay` function have a wrong desing. In this function, are two transfers, the first one is from the user to the transmuter, the repay amount (in yield tokens), and the second one is from the AlchemistV3 contrasctr to the protocolFeeReciver, but this is also transfering the repay amount (in yield tokens). 
Since the contract never receives funds for that use,the funds are being obtained from the collateral deposited by users. But it's not beeing reflected in the account.collateralBalance (storage variable dedicated to the collateral balance deposited by an user).
When the user repay his loan,It is logical that he may wants to withdraw his collateral, but it will end reverting with ERC20InsuficientBalance, because the contract do not have tokens to return all the collateral. 

## Impact Explanation
The user owner of the postion, will lose the amount repaied from the collateral.

## Likelihood Explanation
This will occur all the time a postion debt is repaied.

## Proof of Concept

1. A user deposit 8e18
(alchemix yield token balance 8e18)
2. Get 5e18 debt tokens      
(user's collateral avalible to withdraw 3e18)  
 ---- Pass Some Blocks ----
3. repay the debt of 5e18
(alchemix yield token balance 3e18)
(user's collateral avalible to withdraw 8e18)
4. Try to withdraw all the position's collateral -> this will revert


<details><summary>Proof of Code</summary>
Create a new file with .t.sol extension and paste the following code: 

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.26;

import {Test, console2} from "../../lib/forge-std/src/Test.sol";
import {console} from "../../lib/forge-std/src/console.sol";

import {AlchemistV3} from "../AlchemistV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {Whitelist} from "../utils/Whitelist.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {IAlchemistV3, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";

contract AlchemistV3Test is Test {
    AlchemistV3 alchemist;
    AlchemicTokenV3 alToken;
    TestERC20 fakeUnderlyingToken;
    TestYieldToken fakeYieldToken;

    TransparentUpgradeableProxy proxyAlchemist;
    Transmuter transmuterLogic;
    Whitelist whitelist;
    AlchemistTokenVault alchemistFeeVault;
    AlchemistV3Position alchemistNFT;

    uint256 public minted;
    uint256 public burned;
    uint256 public sentToTransmuter;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;
    uint256 public liquidatorFeeBPS = 300;
    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    uint256 accountFunds = 2_000_000_000e18;
    address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);
    address anotherExternalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;
    address caller;

    function setUp() external {
        caller = address(0xdead);
        address proxyOwner = address(this);

        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        vm.startPrank(caller);

        fakeUnderlyingToken = new TestERC20(100e18, uint8(18));
        fakeYieldToken = new TestYieldToken(address(fakeUnderlyingToken));
        alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        alOwner = caller;
        transmuterLogic = new Transmuter(transParams);
        whitelist = new Whitelist();

        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: address(fakeUnderlyingToken),
            yieldToken: address(fakeYieldToken),
            blocksPerYear: 2_600_000,
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000,
            globalMinimumCollateralization: 1_111_111_111_111_111_111,
            tokenAdapter: address(fakeYieldToken),
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: address(10),
            liquidatorFee: liquidatorFeeBPS
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(new AlchemistV3()), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        alToken.setWhitelist(address(proxyAlchemist), true);
        whitelist.add(address(0xbeef));
        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(address(fakeUnderlyingToken), address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));

        vm.stopPrank();

        deal(address(fakeYieldToken), externalUser, accountFunds);
        deal(address(fakeYieldToken), anotherExternalUser, accountFunds);
        deal(address(alToken), externalUser, accountFunds);
        deal(address(fakeUnderlyingToken), externalUser, accountFunds);
        deal(address(fakeUnderlyingToken), anotherExternalUser, accountFunds);
    }

    function test__RepayTransferFeesFromTheUsersCollateralBalanceProofOfConcept() external {
        vm.startPrank(externalUser);
        // 1. Deposit 8e18 
        fakeYieldToken.approve(address(alchemist), 8e18);
        alchemist.deposit(8e18, externalUser, 0);
        // 2. Get 5 debt tokens (free collateral at this point should be 8 - 5)
        alchemist.mint(1, 5e18, externalUser);
        console.log("Collateral: ",  alchemist.totalValue(1));

        // 3. Go some time in the future
        vm.roll(block.number + 10);
        vm.warp(block.timestamp + 600);

        // 4. Repay the 5e18 debt with more yield tokens
        fakeYieldToken.approve(address(alchemist), 5e18);
        console2.log("Balnce before of fee of alchemist is: ", fakeYieldToken.balanceOf(address(alchemist)));
        uint256 amount = alchemist.repay(5e18, 1);

        console2.log("Balnce after of fee of alchemist is: ", fakeYieldToken.balanceOf(address(alchemist)));
        console2.log("Fee, and transfer amount is: ", amount);


        //5. Check that the free collateral, avalible to withdraw of the user is enough and withdraw it
        console.log("Collateral: ",  alchemist.totalValue(1));
        vm.expectRevert();
        alchemist.withdraw(8000000000000000000, externalUser, 1);
        //6. Reverts because when the repay function transfer to the protocolFeeReceiver, the contract is using the money deposited as collateral by the users

        vm.stopPrank();
    }
}

```

To run the test just run in the console: 

```bash
     forge test --mt test__RepayTransferFeesFromTheUsersCollateralBalanceProofOfConcept -vvv
```
</details>

## Recommendation

1. Calculate the fees, not transfering the whole repaing amount to the fee reciver.
2. Get fees from user's yield token transfer amount, not from the contract balance.

```diff
+     uint256 feeAmount = creditToYield * protocolFee / BPS;
-      TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, creditToYield);
+     TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, feeAmount );
``` 


