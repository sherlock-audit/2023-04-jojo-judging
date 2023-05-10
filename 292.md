ltyu

high

# Unsafe reserve delisting can cause liquidations

## Summary
JoJo admin can delist reserve tokens, which can be detrimental if the delisted token is being used as collateral. When a token is delisted, its collateral value is no longer included in the total sum of collateral (`liquidationMaxMintAmount`), which can result in account liquidation.

## Vulnerability Detail
In JUSDBank, the function `liquidate` calls `_isStartLiquidation` to determine if a user can be liquidated. In essence, `_isStartLiquidation` sums up the total value of all reserve tokens deposited. 
```solidity
function _isStartLiquidation(
        DataTypes.UserInfo storage liquidatedTraderInfo,
        uint256 tRate
    ) internal view returns (bool) {
        uint256 JUSDBorrow = (liquidatedTraderInfo.t0BorrowBalance).decimalMul(
            tRate
        );
        uint256 liquidationMaxMintAmount;
        address[] memory collaterals = liquidatedTraderInfo.collateralList;
        for (uint256 i; i < collaterals.length; i = i + 1) {
            address collateral = collaterals[i];
            DataTypes.ReserveInfo memory reserve = reserveInfo[collateral];
            if (reserve.isFinalLiquidation) {
                continue; // @audit value is 0 if delisted
            }
            liquidationMaxMintAmount += _getMintAmount(
                liquidatedTraderInfo.depositBalance[collateral],
                reserve.oracle,
                reserve.liquidationMortgageRate
            );
        }
        return liquidationMaxMintAmount < JUSDBorrow;
    }	
```
If the user's borrowed JUSD is greater than their `liquidationMaxMintAmount`, they are at risk of being liquidated. In some cases, if a reserve has been delisted, its value will be considered effectively zero, which means that the token's actual value will not be counted in the user's total collateral value.

This issue can be illustrated by having 2 reserve tokens:
1. Alice deposits equal amounts of Token A and Token B. Lets say their value are also equal. Her `liquidationMaxMintAmount` is 100 (50 for each).
2. Alice mints 75 JUSD, her `JUSDBorrow` will be 75.
3. Token A gets delisted
4. Since, the value of Token A is considered 0, her `liquidationMaxMintAmount` will be 50. Token B can be liquidated.

## Proof of Concept
A unit test is created to show that Alice is safe from liquidations. Once a reserve token has been removed, Alice can be liquidated. Add this test to `JUSDBankLiquidateCollateral.t.sol`
```solidity
function testLiquidateDelist() public {
        // Setup, deposit 10e16 mockToken1 and deposit 1e8 mockToken2, borrow 8000e6 
        mockToken1.transfer(alice, 10e18);
        mockToken2.transfer(alice, 10e8);
        vm.prank(address(jusdBank));
        jusd.transfer(bob, 5000e6);

        vm.startPrank(alice);
        mockToken1.approve(address(jusdBank), 10e18);
        mockToken2.approve(address(jusdBank), 10e8);
        jusdBank.deposit(alice, address(mockToken1), 10e18, alice);
        jusdBank.borrow(8000e6, alice, false); // max amount borrowable in terms of mockeToken1. Price is 100ke6.
        vm.stopPrank();

        // Drop price to 90ke6
        vm.startPrank(address(this));
        MockChainLink900 eth900 = new MockChainLink900(); 
        JOJOOracleAdaptor jojoOracle900 = new JOJOOracleAdaptor(
            address(eth900),
            20,
            86400,
            address(usdcPrice)
        );
        jusdBank.updateOracle(address(mockToken1), address(jojoOracle900));
        swapContract.addTokenPrice(address(mockToken1), address(jojoOracle900));
        vm.stopPrank();

        assertEq(jusdBank.isAccountSafe(alice), false);
        vm.prank(alice);
        jusdBank.deposit(alice, address(mockToken2), 1e8, alice); // Account is now safe
        assertEq(jusdBank.isAccountSafe(alice), true);


        vm.startPrank(bob);
        vm.warp(3000);
        uint256 token1AmountToLiquidate = 10e18; // all of it
        bytes memory data = swapContract.getSwapData(token1AmountToLiquidate, address(mockToken1));
        bytes memory param = abi.encode(
            swapContract,
            swapContract,
            address(bob),
            data
        );
        FlashLoanLiquidate flashloanRepay = new FlashLoanLiquidate(
            address(jusdBank),
            address(jusdExchange),
            address(USDC),
            address(jusd),
            insurance
        );
        bytes memory afterParam = abi.encode(address(flashloanRepay), param);
        
        vm.expectRevert("ACCOUNT_IS_SAFE");
        jusdBank.liquidate(
            alice,
            address(mockToken1),
            bob,
            token1AmountToLiquidate,
            afterParam,
            10e18
        );
        vm.stopPrank();

        // Makes account liquidable by removing mockToken2
        jusdBank.delistReserve(address(mockToken2));

        // Liquidate mockToken1
        vm.prank(bob);
        jusdBank.liquidate(
            alice,
            address(mockToken1),
            bob,
            token1AmountToLiquidate,
            afterParam,
            10e18
        );
}
```

## Impact
Reserve delisting can cause user liquidations.

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L379-L392

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDView.sol#L182-L204
## Tool used

Manual Review

## Recommendation
Consider adding safety measures to prevent delisting of reserve token used for collateral. If it is essential that the protocol value a delisted token as 0, consider only allowing liquidations up to the delisted token value.
