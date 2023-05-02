BenRai

high

# If user has to many open positions, the function `getTotalExposure()` will fail because it runs out of gas

## Summary

If a user has to many open positions the function `getTotalExposure()` will run out of gas while calculating the `netPositionValue` and looping over ` openPositions[trader]`. This will result in the failure of all functions that call the function `getTotalExposure()`.

## Vulnerability Detail

The functions ` _isSafe ` and `_isSolidSafe` are calling `getTotalExposure()` and can therefore run out of gas. 

 The following “end functions” are affected by this and will also run out of gas:

`Perpetual.trade()`,
`Perpetual. liquidate()` (trough `Liquidation.handleBadDebt()`),
‘Funding. executeWithdraw()’ (through `Funding. _withdraw`),
`Liquidation.requestLiquidation()` (trough `Liquidation.getLiquidateCreditAmount()`)
`JOJOExternal. approveTrade()`



## Impact

A user that has to many open positions can not withdraw any assets even if he would meet the necessary criteria.  (‘Funding. executeWithdraw()’ is failing)  He would be forced to close positions to free his assets again by using the function `_realizePnl`. Since this function `_realizePnl` also runs a loop over all open Positions of the user  to find the position that should be closed, the user would need to close one of his oldest positions to avoid running out of gas here also. If the user has eg. 100 positions open and he would need e.g. 90 to be able withdraw his assets he would be forced to close 10 positions. If the positions that he can close without running out of gas using the function `_realizePnl`  are all out of profit, he might be forced to realize loses. This might lead to him leaving the JOJO ecosystem. This is especially bad for the exchange since individuals who have a lot of open positions are the once that trade a lot and bring in the most fee revenues.


## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/6090fef68932b5577abf6b5aa26eb1e579353c57/smart-contract-EVM/contracts/lib/Liquidation.sol#L63-L84


## Tool used

Manual Review

## Recommendation

Set a cap of open positions one address can have. If the user wands to open more positions he can create a sub account and open new positions there. 
