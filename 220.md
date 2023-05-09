J4de

medium

# `JUSDView.sol#_getMintAmountBorrow`  should be divided by the token's decimal instead of the fixed value `1e18`

## Summary

`JUSDView.sol#_getMintAmountBorrow`  should be divided by the token's decimal instead of the fixed value `1e18`

## Vulnerability Detail

```solidity
File: Impl/JUSDView.sol
106     function _getMintAmountBorrow(
107         DataTypes.ReserveInfo memory reserve,
108         uint256 amount
109     ) internal view returns (uint256) {
110         uint256 depositAmount = IPriceChainLink(reserve.oracle)
111             .getAssetPrice()
112 >>          .decimalMul(amount)
113             .decimalMul(reserve.initialMortgageRate);
114         if (depositAmount >= reserve.maxColBorrowPerAccount) {
115             depositAmount = reserve.maxColBorrowPerAccount;
116         }
117         return depositAmount;
118     }
```

The `_getMintAmountBorrow` function is used to calculate the value of despoit, and the `depositAmount` is the value of despoit tokens, and its decimal should be the same as the decimal of price (1e18). Therefore, when the amount of the token is multiplied by the price of the token, it should be divided by the decimal of the token instead of the decimal of the price (fixed to 1e18).

The same problem happens with `Liquidation.sol#getTotalExposure`, `Liquidation.sol#getLiquidateCreditAmount`.

## Impact

May lead to miscalculation of value

## Code Snippet

https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDView.sol#L112

https://github.com/JOJOexchange/smart-contract-EVM/blob/4a95a8e9a6367ae88dc827e29467229cb5bbad4f/contracts/lib/Liquidation.sol#L75

https://github.com/JOJOexchange/smart-contract-EVM/blob/4a95a8e9a6367ae88dc827e29467229cb5bbad4f/contracts/lib/Liquidation.sol#L320

## Tool used

Manual Review

## Recommendation

It is recommended to use token's decimal

```diff
    function _getMintAmountBorrow(
        DataTypes.ReserveInfo memory reserve,
        uint256 amount
    ) internal view returns (uint256) {
        uint256 depositAmount = IPriceChainLink(reserve.oracle)
            .getAssetPrice()
-           .decimalMul(amount)
+           .mul(amount).div(reserve.token.decimals()) // just a demo
            .decimalMul(reserve.initialMortgageRate);
        if (depositAmount >= reserve.maxColBorrowPerAccount) {
            depositAmount = reserve.maxColBorrowPerAccount;
        }
        return depositAmount;
    }
```