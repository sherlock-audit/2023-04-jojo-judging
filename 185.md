Bauer

medium

# The insurance account miss check for maxPerAccountBorrowAmount

## Summary

The insurance account miss check for maxPerAccountBorrowAmount
## Vulnerability Detail
The `JUSDBank._handleBadDebt()`  is used to handle bad debt resulting from the liquidation of a trader's debt, where the debt cannot be fully recovered from the sale of the trader's collateral.The function updates the t0BorrowBalance of the insuranceInfo UserInfo object by adding the borrowJUSDT0 (the amount of JUSD debt in t0 tokens) that was not able to be recovered from the liquidation.
According to the document  there is a maximum amount of funds that an individual account is allowed to borrow from  protocol. However, there is no check for insurance account.
```solidity
   function _handleBadDebt(address liquidatedTrader) internal {
        DataTypes.UserInfo storage liquidatedTraderInfo = userInfo[
            liquidatedTrader
        ];
        uint256 tRate = getTRate();
        if (
            liquidatedTraderInfo.collateralList.length == 0 &&
            _isStartLiquidation(liquidatedTraderInfo, tRate)
        ) {
            DataTypes.UserInfo storage insuranceInfo = userInfo[insurance];
            uint256 borrowJUSDT0 = liquidatedTraderInfo.t0BorrowBalance;
            insuranceInfo.t0BorrowBalance += borrowJUSDT0;
            liquidatedTraderInfo.t0BorrowBalance = 0;
            emit HandleBadDebt(liquidatedTrader, borrowJUSDT0);
        }
    }

```
## Impact
The insurance account miss check for maxPerAccountBorrowAmount
## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L496-L511
## Tool used

Manual Review

## Recommendation
add check:
```solidity
      require(
            user.t0BorrowBalance.decimalMul(tRate) <= maxPerAccountBorrowAmount,
            JUSDErrors.EXCEED_THE_MAX_BORROW_AMOUNT_PER_ACCOUNT
        );
```
