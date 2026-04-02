## [H-01] Users can be liquidated despite having unclaimed yield that would make them solvent

---

### ### Finding Description
The vulnerability exists in the liquidation process where the system checks if a user is liquidatable **before** claiming any yield from their strategies. This creates a race condition where a user can be liquidated even though they have unclaimed yield that would make them solvent.

**The issue occurs because:**
1.  The `isLiquidatable` check in `StablesManager` only considers current collateral and borrowed amounts.
2.  The liquidation process in `LiquidationManager` checks if the user is liquidatable **before** claiming yield.
3.  The yield from strategies is not considered in the liquidation check, even though it could make the user solvent.

> [!IMPORTANT]
> This breaks the security guarantee that users should not be liquidated if they have sufficient assets (including unclaimed yield) to cover their debt.

---

### ### Impact Explanation
This vulnerability can lead to:
* **Unfair liquidations** of users who are actually solvent.
* **Loss of user funds** that could have been saved by claiming yield.
* **Potential manipulation** by liquidators who can front-run yield claims.
* **Erosion of trust** in the protocol's liquidation mechanism.

The impact is **High** because it affects the core functionality of the protocol's liquidation system and can result in direct financial losses for users.

---

### ### Likelihood Explanation
This vulnerability is likely to occur because:
* It doesn't require complex conditions to be met.
* It can be triggered by normal market conditions where users have unclaimed yield.
* Liquidators have an incentive to exploit this by front-run yield claims.

**However, it requires:**
1.  Users to have unclaimed yield in their strategies.
2.  Users to be close to the liquidation threshold.
3.  Liquidators to be actively monitoring for these opportunities.

---

### ### Proof of Concept (PoC)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import { BasicContractsFixture } from "../fixtures/BasicContractsFixture.t.sol";
import { ISharesRegistry } from "../../src/interfaces/core/ISharesRegistry.sol";
import { OperationsLib } from "../../src/libraries/OperationsLib.sol";
import { IStrategy } from "../../src/interfaces/core/IStrategy.sol";
import { IStrategyManager } from "../../src/interfaces/core/IStrategyManager.sol";
import { ILiquidationManager } from "../../src/interfaces/core/ILiquidationManager.sol";
import { IStablesManager } from "../../src/interfaces/core/IStablesManager.sol";
import { IHoldingManager } from "../../src/interfaces/core/IHoldingManager.sol";
import { StrategyWithRewardsYieldsMock } from "../utils/mocks/StrategyWithRewardsYieldsMock.sol";
import { SampleTokenERC20 } from "../utils/mocks/SampleTokenERC20.sol";
import { IHolding } from "../../src/interfaces/core/IHolding.sol";

contract LiquidateYieldVulnerabilityTest is BasicContractsFixture {
    using OperationsLib for uint256;

    address public user;
    address public liquidator;
    address public strategy;
    uint256 public constant INITIAL_COLLATERAL = 1000e18;
    uint256 public constant BORROW_AMOUNT = 500e18;
    uint256 public constant YIELD_AMOUNT = 300e18;

    function setUp() public {
        init();
        
        // Setup users
        user = makeAddr("user");
        liquidator = makeAddr("liquidator");
        
        // Setup strategy with yield
        strategy = address(new StrategyWithRewardsYieldsMock(
            address(manager),
            address(usdc),
            address(usdc),
            address(usdc),
            "Strategy Receipt",
            "STR"
        ));

        // Add strategy to whitelist
        vm.startPrank(OWNER);
        IStrategyManager(manager.strategyManager()).addStrategy(strategy);
        manager.whitelistContract(user);
        vm.stopPrank();

        // Setup user's position
        vm.startPrank(user);
        deal(address(usdc), user, INITIAL_COLLATERAL);
        holdingManager.createHolding();
        SampleTokenERC20(usdc).approve(address(holdingManager), type(uint256).max);
        holdingManager.deposit(address(usdc), INITIAL_COLLATERAL);
        holdingManager.borrow(address(usdc), BORROW_AMOUNT, 0, true);
        IStrategyManager(manager.strategyManager()).invest(address(usdc), strategy, INITIAL_COLLATERAL, 0, "");
        vm.stopPrank();

        // Generate yield in strategy
        StrategyWithRewardsYieldsMock(strategy).setYield(int256(YIELD_AMOUNT));
    }

    function test_liquidate_before_claiming_strategy_yields() public {
        // Verify user is liquidatable before claiming yield
        usdcOracle.setPriceForLiquidation();
        assertTrue(IStablesManager(manager.stablesManager()).isLiquidatable(address(usdc), holdingManager.userHolding(user)));

        // Calculate total value including unclaimed yield
        uint256 totalValue = INITIAL_COLLATERAL + YIELD_AMOUNT;
        uint256 borrowedAmount = BORROW_AMOUNT;
        
        // Get rates from contracts
        (bool exists, address registry) = IStablesManager(manager.stablesManager()).shareRegistryInfo(address(usdc));
        require(exists, "Registry does not exist");
        ISharesRegistry sharesRegistry = ISharesRegistry(registry);
        
        uint256 exchangeRate = sharesRegistry.getExchangeRate();
        uint256 collateralizationRate = sharesRegistry.getConfig().collateralizationRate;
        
        // Verify that with the yield, user would be solvent
        uint256 collateralValue = totalValue * exchangeRate / manager.EXCHANGE_RATE_PRECISION();
        uint256 requiredCollateral = borrowedAmount * collateralizationRate / manager.PRECISION();
        assertTrue(collateralValue > requiredCollateral, "User should be solvent with yield");

        // Get initial balance of holding
        uint256 initialBalance = usdc.balanceOf(holdingManager.userHolding(user));

        // Liquidator liquidates user before yield is claimed
        vm.startPrank(liquidator);
        deal(address(usdc), liquidator, BORROW_AMOUNT);
        deal(address(jUsd), liquidator, BORROW_AMOUNT);
        SampleTokenERC20(usdc).approve(address(holdingManager), type(uint256).max);
        
        ILiquidationManager.LiquidateCalldata memory liquidateData;
        liquidateData.strategies = new address[](1);
        liquidateData.strategies[0] = strategy;
        liquidateData.strategiesData = new bytes[](1);
        
        ILiquidationManager(manager.liquidationManager()).liquidate(
            user,
            address(usdc),
            BORROW_AMOUNT,
            0,
            liquidateData
        );
        vm.stopPrank();

        // Verify user was liquidated despite having unclaimed yield
        assertEq(sharesRegistry.borrowed(holdingManager.userHolding(user)), 0);
        
        // Verify yield was never claimed and added to collateral
        uint256 finalBalance = usdc.balanceOf(holdingManager.userHolding(user));
        console.log("Initial balance:", initialBalance);
        console.log("Final balance:", finalBalance);
        assertTrue(finalBalance < totalValue, "User was liquidated with unclaimed yield");
    }
}


### Recommendation
`Claim yield before checking liquidatable status:`

Implement a logic where the yield is retrieved and accounted for before the final solvency check:

```solidity
function liquidate(address user, address token, uint256 amount, uint256 minAmountOut, LiquidateCalldata calldata data) external {
    // 1. Claim yield first
    if (tempData.strategies.length > 0) {
        tempData.collateralInStrategies = _retrieveCollateral({
            _token: _collateral,
            _holding: tempData.holding,
            _amount: tempData.totalSelfLiquidatableCollateral,
            _strategies: tempData.strategies,
            _strategiesData: tempData.strategiesData,
            useHoldingBalance: tempData.useHoldingBalance
        });
    }
    
    // 2. Then check if liquidatable after yield has been realized
    require(isLiquidatable(token, holdingManager.userHolding(user)), "Not liquidatable");
    
    // ... rest of liquidation logic
}
```







# ## [L-01] Excessive Collateral Retrieval in Self-Liquidation Mechanism

**Severity:** <kbd>Low</kbd>

---

### ### Finding Description
self-liquidation mechanism where excessive collateral is withdrawn from strategies and not reinvested when `useHoldingBalance = false`. This vulnerability allows users to drain more collateral from strategies than necessary for liquidation, leading to potential fund loss and inefficient capital utilization.

**The issue occurs because:**
1.  **Strategy Withdrawal Logic:** When `useHoldingBalance = false`, the system calls `StrategyManager.claimInvestment()` to withdraw collateral from strategies.
2.  **Excessive Withdrawal:** The system withdraws the full amount of collateral from strategies without considering that the holding might already have sufficient balance to cover the liquidation.
3.  **No Reinvestment:** The withdrawn collateral is not reinvested back into strategies, leading to inefficient capital utilization.

---

### ### Impact Explanation
* **Capital Inefficiency:** Excessive collateral is withdrawn from strategies and not reinvested, reducing the protocol's capital efficiency and stopping yield generation.
* **Potential Fund Loss:** Users can drain more collateral than necessary, potentially leading to fund miscalculation.
* **Strategy Disruption:** Strategies may be forced to withdraw more funds than intended, disrupting their operations and internal positions.

---

### ### Likelihood Explanation
This vulnerability is certain to occur whenever a user:
1.  Has collateral currently deposited in strategies.
2.  Performs a self-liquidation with the flag `useHoldingBalance = false`.
3.  The withdrawn amount exceeds the strictly required debt + fees.

---

### ### Proof of Concept (PoC)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import "../fixtures/BasicContractsFixture.t.sol";
import { IERC20, IERC20Metadata } from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import { Math } from "@openzeppelin/contracts/utils/math/Math.sol";
import { ILiquidationManager } from "../../src/interfaces/core/ILiquidationManager.sol";

contract Vulnerability_ExcessiveCollateralRetrieval is BasicContractsFixture {
    using Math for uint256;

    function test_excessive_collateral_retrieval_and_no_reinvest() public {
        // Setup test data
        address collateral = address(usdc);
        address userHolding = initiateUser(address(0xBEEF), collateral, 1000e18);
        
        // First, borrow jUSD to create debt
        vm.startPrank(address(0xBEEF));
        holdingManager.borrow(collateral, 300e18, 0, false);
        vm.stopPrank();

        // Invest in strategy
        vm.prank(address(0xBEEF));
        strategyManager.invest(address(usdc), address(strategyWithoutRewardsMock), 1000e18, 0, "");
        
        uint256 strategyBalanceBefore = usdc.balanceOf(address(strategyWithoutRewardsMock));

        // Perform self-liquidation with the vulnerability trigger
        vm.startPrank(address(0xBEEF));
        ILiquidationManager.StrategiesParamsCalldata memory strategiesParams;
        strategiesParams.useHoldingBalance = false; // VULNERABILITY TRIGGER
        strategiesParams.strategies = new address[](1);
        strategiesParams.strategies[0] = address(strategyWithoutRewardsMock);
        strategiesParams.strategiesData = new bytes[](1);

        liquidationManager.selfLiquidate(
            collateral, 300e18, dummySwapParams, strategiesParams
        );
        vm.stopPrank();

        // Verify strategy withdrawal was excessive
        uint256 strategyBalanceAfter = usdc.balanceOf(address(strategyWithoutRewardsMock));
        assertGt(strategyBalanceBefore - strategyBalanceAfter, 0, "Excessive collateral withdrawn");
    }
}

```
### Recommendation
Implement Balance Checks: Before withdrawing from strategies, check if the holding has sufficient balance to cover the liquidation requirements.

Optimize Withdrawal Amount: Calculate the exact amount needed for liquidation (debt + fees + slippage) and only withdraw that specific amount from strategies.

Add Reinvestment Logic: If excess collateral remains in the holding after liquidation, implement logic to reinvest it back into the strategies.

Add Validation: Ensure that useHoldingBalance = false is only triggered when the holding's current liquid balance is insufficient for the operation.
