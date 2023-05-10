cccz

medium

# In over liquidation, if the liquidatee has USDC-denominated assets for sale, the liquidator can buy the assets with USDC to avoid paying USDC to the liquidatee

## Summary
In over liquidation, if the liquidatee has USDC-denominated assets for sale, the liquidator can buy the assets with USDC to avoid paying USDC to the liquidatee
## Vulnerability Detail
In JUSDBank contract, if the liquidator wants to liquidate more collateral than the borrowings of the liquidatee, the liquidator can pay additional USDC to get the liquidatee's collateral. 
```solidity
        } else {
            //            actualJUSD = actualCollateral * priceOff
            //            = JUSDBorrowed * priceOff / priceOff * (1-insuranceFeeRate)
            //            = JUSDBorrowed / (1-insuranceFeeRate)
            //            insuranceFee = actualJUSD * insuranceFeeRate
            //            = actualCollateral * priceOff * insuranceFeeRate
            //            = JUSDBorrowed * insuranceFeeRate / (1- insuranceFeeRate)
            liquidateData.actualCollateral = JUSDBorrowed
                .decimalDiv(priceOff)
                .decimalDiv(JOJOConstant.ONE - reserve.insuranceFeeRate);
            liquidateData.insuranceFee = JUSDBorrowed
                .decimalMul(reserve.insuranceFeeRate)
                .decimalDiv(JOJOConstant.ONE - reserve.insuranceFeeRate);
            liquidateData.actualLiquidatedT0 = liquidatedInfo.t0BorrowBalance;
            liquidateData.actualLiquidated = JUSDBorrowed;
        }

        liquidateData.liquidatedRemainUSDC = (amount -
            liquidateData.actualCollateral).decimalMul(price);
```
The liquidator needs to pay USDC in the callback and the JUSDBank contract will require the final USDC balance of the liquidatee to increase.
```solidity
        require(
            IERC20(primaryAsset).balanceOf(liquidated) -
                primaryLiquidatedAmount >=
                liquidateData.liquidatedRemainUSDC,
            JUSDErrors.LIQUIDATED_AMOUNT_NOT_ENOUGH
        );
```
If the liquidatee has USDC-denominated assets for sale, the liquidator can purchase the assets with USDC in the callback, so that the liquidatee's USDC balance will increase and the liquidator will not need to send USDC to the liquidatee to pass the check in the JUSDBank contract.
## Impact
In case of over liquidation, the liquidator does not need to pay additional USDC to the liquidatee
## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L188-L204

## Tool used

Manual Review

## Recommendation
Consider banning over liquidation
