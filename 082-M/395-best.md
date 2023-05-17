GalloDaSballo

high

# Attacker can prevent `FlashLoanLiquidate` by buying all JUSD from JUSDExchange

## Summary

`FlashLoanLiquidate` relies on `IJUSDExchange.buyJUSD` to obtain JUSD

If all JUSD is bought, no liquidation can be performed

By front-running a liquidation and buying all JUSD, liquidations can be prevented


## Vulnerability Detail

This can be done as a sandwich by:
- Borrowing the amount of JUSD that will be bought, and selling it for USDC on a DEX
- Buying all JUSD available from the JUSDExchange
- Preventing the Liquidation
- Repaying the debt, unlocking all collateral and avoiding the liquidation

This can also be methodically be performed by just buying all JUSD from the USDExchange and then selling it on a CEX liquidity pool, breaking liquidation script

## Impact

Liquidations performed via the official contract can always be denied

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/flashloanImpl/FlashLoanLiquidate.sol#L75-L78

## Tool used

Manual Review

## Recommendation

Allow buying of JUSD from other sources in `FlashLoanLiquidate` or change JUSDExchange to allow flashMinting of JUSD
