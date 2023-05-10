immeas

medium

# `JOJOBank::handleBadDept` can be blocked with dust to keep debt bad

## Summary
When clearing bad debt, a check is done that there is no collateral. This could be blocked by adding a dust amount of collateral to prevent clearing the bad debt. Which could be beneficial since it could reduce trust and lower the price of JUSD.

## Vulnerability Detail
The protocol can chose to move bad debt over to `insurance`. This can only be done when there is no collateral backing the bad debt:

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L501-L510
```solidity
File: JUSDV1/src/Impl/JUSDBank.sol

501:        if (
502:            liquidatedTraderInfo.collateralList.length == 0 &&
503:            _isStartLiquidation(liquidatedTraderInfo, tRate)
504:        ) {
505:            DataTypes.UserInfo storage insuranceInfo = userInfo[insurance];
506:            uint256 borrowJUSDT0 = liquidatedTraderInfo.t0BorrowBalance;
507:            insuranceInfo.t0BorrowBalance += borrowJUSDT0;
508:            liquidatedTraderInfo.t0BorrowBalance = 0;
509:            emit HandleBadDebt(liquidatedTrader, borrowJUSDT0);
510:        }
```

This moves the bad debt over to `insurance` where it [potentially can be handled](https://github.com/sherlock-audit/2023-04-jojo-0ximmeas/issues/9).

This requires there to be no collateral at all. A malicious user could block this by adding dust amounts of collateral. Since this fails silently, i.e. the transaction will not revert it could go unnoticed as well.

Since it just dust it is not worth the gas liquidating this.

## Impact
Bad debt is very dangerous for the protocol as that could build up and lower the price of JUSD. Since JUSD is accepted collateral in the JOJO market, a lower price for JUSD will let you open positions with less backing.

If there is a large drop in the value of a collateral which creates a lot of bad dept this could be an issue.

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#LL502C34-L502C48

## Tool used
Manual Review

## Recommendation
Add a minimum deposit amount to make this attack more costly. If the minimum deposit amount is more than the gas price of liqudating performing this attack is just giving liquidators the collateral (unless the account is made safe again which is also good).