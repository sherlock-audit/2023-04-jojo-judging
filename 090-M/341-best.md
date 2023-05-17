branch_indigo

medium

# JUSD Liquidate ExpectPrice Comparison Not Properly Scaled When Collateral is WBTC

## Summary
In JUSDBank.sol `liquidate`, the comparison to determine whether a liquidator input expectPrice is reached is not properly scaled, might potentially resulting all liquidation reverts. 
## Vulnerability Detail
```solidity
//JUSDBank.sol-liquidate()
        require(
            (liquidateData.insuranceFee + liquidateData.actualLiquidated)
                .decimalDiv(liquidateData.actualCollateral) <= expectPrice,
            JUSDErrors.LIQUIDATION_PRICE_PROTECTION
        );
```
In `liquidate`, the expectPrice is input by liquidators, which is ratio between primary token assets and collateral token assets. The current scaling on the left side results in different decimal places when the collateral token decimal places are different, this effectively requires inconsistencies in expectPrice decimal places, possibly lead to liquidate reverts.

InsuranceFee and actualLiquidated are in 6 decimals following in primary asset token decimals. When collateral token is in 18 decimals (WETH), the ratio is in 6 decimals; When collateral token is in 8 decimals (WBTC), the ratio is in 16 decimals. This discrepancy in decimal places is not handled.

It should be noted while price feed from JOJOOracleAdaptor is in different decimals depending on collateral token decimals which was later adjusted during internal collateral calculations. The same shouldn't be expected for liquidators who are public users with no knowledge on internal JOJO price decimal conversions. So current comparison implementations would most likely result in liquidation reverts, especially when an idiosyncratic 16 decimal ratio value is needed when the collateral is WBTC.

## Impact
Liquidations might revert

## Code Snippet
[https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDBank.sol#L171-L177](https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDBank.sol#L171-L177)


## Tool used

Manual Review

## Recommendation
Consider properly handle decimals on the left side of the comparison, so no inconsistency on expectPrice decimal places.