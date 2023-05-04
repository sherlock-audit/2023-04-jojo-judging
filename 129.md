Nyx

high

# If JUSD depegs , borrowers can get liquated unfairly.

## Summary
If JUSD depegs from its pegged value (1 JUSD = 1 USDC), the function still considers JUSD equal to 1 USDC, which might put the borrower in an unfavorable position.
## Vulnerability Detail
When a user tries to liquidate another user, the _isStartLiquidation() function checks whether the borrower is liquitable or not.
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
                continue;
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
But the _isStartLiquidation() function assumes JUSD always equals 1 USD(or USDC).
```solidity
// JUSDBorrow = (liquidatedTraderInfo.t0BorrowBalance).decimalMul(tRate);
return liquidationMaxMintAmount < JUSDBorrow; 
```
If JUSD depegs and _isStartLiquidation() assumes JUSD = 1, there will be earlier liquidations than it should be.
```solidity
// JUSD depegs and lets say equals 0.9
// But let's do the calculation with JUSD = 1
// liquidationMaxMintAmount = 950
// JUSDBorrow = 1000 

return liquidationMaxMintAmount < JUSDBorrow
// returns true, and the borrower gets liquidated.
```
```solidity
// JUSD depegs and let's say equals 0.9
// This time, let's calculate with JUSD = 0.9.
// liquidationMaxMintAmount = 950
// JUSDBorrow = 900 

return liquidationMaxMintAmount < JUSDBorrow
// returns false, the borrower is safe.
```

## Impact
Borrowers get liquidated early.
## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDView.sol#L179-L204
## Tool used

Manual Review

## Recommendation
When retrieving the borrower's balance, consider obtaining the JUSD price from the oracle and use it to calculate the JUSDBorrow.