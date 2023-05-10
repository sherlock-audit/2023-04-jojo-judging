J4de

medium

# `JUSDBank.sol#_calculateLiquidateAmount` user can front-run liquidate 1 token to prevent others from liquidating

## Summary

`JUSDBank.sol#_calculateLiquidateAmount` user can front-run repay by 1 token to prevent others from liquidating

## Vulnerability Detail

```solidity
File: Impl/JUSDBank.sol
379     function _calculateLiquidateAmount(
380         address liquidated,
381         address collateral,
382         uint256 amount
383     ) internal view returns (DataTypes.LiquidateData memory liquidateData) {
384         DataTypes.UserInfo storage liquidatedInfo = userInfo[liquidated];
385         require(amount != 0, JUSDErrors.LIQUIDATE_AMOUNT_IS_ZERO);
386         require(
387 >>          amount <= liquidatedInfo.depositBalance[collateral],
388             JUSDErrors.LIQUIDATE_AMOUNT_IS_TOO_BIG
389         );
```

If the liquidation amount is greater than the user's collateral amount during liquidation, revert will occur. Users can expoit this to prevent liquidation:

1. Bob has 100 * 1e18 ETH for collateral, and the position is liquidable
2. Alias is going to liquidate all of Bob's ETH, a total of 100 * 1e18
3. Bob discovered Alias' request, and the front-run liquidated 1 ETH
4. Liquidation fails because 100 * 1e18 > (100 * 1e18 - 1)

P.S. This prevents liquidation itself is invalid, because anyone can have unlimited addresses

```solidity
        require(
            liquidator != liquidated,
            JUSDErrors.SELF_LIQUIDATION_NOT_ALLOWED
        );
```

## Impact

Users can expoit it to prevent being liquidated

## Code Snippet

https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDBank.sol#L387

## Tool used

Manual Review

## Recommendation

It is recommended to deal with the maximum value when it exceeds

```diff
    function _calculateLiquidateAmount(
        address liquidated,
        address collateral,
        uint256 amount
    ) internal view returns (DataTypes.LiquidateData memory liquidateData) {
        DataTypes.UserInfo storage liquidatedInfo = userInfo[liquidated];
        require(amount != 0, JUSDErrors.LIQUIDATE_AMOUNT_IS_ZERO);
-       require(
-           amount <= liquidatedInfo.depositBalance[collateral],
-           JUSDErrors.LIQUIDATE_AMOUNT_IS_TOO_BIG
-       );
+       if (amount > liquidatedInfo.depositBalance[collateral]) {
+           amount = liquidatedInfo.depositBalance[collateral];
+       }
```