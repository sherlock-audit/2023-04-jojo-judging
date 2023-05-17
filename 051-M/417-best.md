GalloDaSballo

medium

# JUSD will depeg with `primaryAsset` (USDC)

## Summary

Because `JUSDExchange` offers a 1-1 exchange between `primaryAsset` (USDC) and `JUSD`, in the case of a depeg, JUSD will also depeg.

## Vulnerability Detail

Due to the accepted 1-1 rate, if USDC becomes cheaper, JUSD will be rapidly arbitraged down to the depeg value.

USDCs price will drag JUSD down with it.

Because JUSD doesn't have redemptions, there will be no immediate relief to the price

The only time in which JUSD will move back to it's $1 price is during liquidations

## Impact

The `owner` will take a loss for the difference between the "face" value of JUSD and the actual value of USDC

During the last depeg this was over 13% of discount
<img width="817" alt="Screenshot 2023-05-10 at 15 15 20" src="https://github.com/sherlock-audit/2023-04-jojo-GalloDaSballo/assets/13383782/877c6ee7-da47-4d82-ad9a-81699fb657c0">

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDExchange.sol#L41-L46
1-1 swap means depeged USDC will be acceptd

## Tool used

Manual Review

## Recommendation

Consider the depeg scenario, add circuit breakers, an oracle, a fee, or accept the risk and understand that a loss will happen