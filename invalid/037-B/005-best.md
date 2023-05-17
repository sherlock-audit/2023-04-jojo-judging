climber2002

medium

# `ReserveInfo.totalDepositAmount` isn't meaningful when there are multiple collateral tokens

## Summary
`ReserveInfo.totalDepositAmount` sums up  amount of different deposited tokens, such as USDC, WETH, WBTC etc. However those tokens have different prices and decimals, so it's not very meaningful.

## Vulnerability Detail
In [JUSDBank._deposit](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L260), it adds `amount` to `reserve.totalDepositAmount`, however the corresponding collateral token could be any token that is allowed, and each token has different prices and decimals. So `reserve.totalDepositAmount` isn't meaningful to contains amount of different tokens.

## Impact
This metric isn't meaningful as it contains amount summed up of different collateral tokens. 

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L260

## Tool used

Manual Review

## Recommendation
Use a mapping `(address => uint256)` to contains the amount of different tokens.