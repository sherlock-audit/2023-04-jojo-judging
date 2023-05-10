Ace-30

high

# Smart contract EVM: Malicious user can prevent liquidation by openning lots of positions due to gas limit

## Summary
to liquidate an account one should call requestLiquidation()

- requestLiquidation() calls getLiquidateCreditAmount()
- getLiquidateCreditAmount() calls _isSafe()
- _isSafe() calls getTotalExposure()
- getTotalExposure() goes through all openPositions of trader
```solidity
    function getTotalExposure(Types.State storage state, address trader)
        public
        view
        returns (
            int256 netPositionValue,
            uint256 exposure,
            uint256 maintenanceMargin
        )
    {
        // sum net value and exposure among all markets
        for (uint256 i = 0; i < state.openPositions[trader].length; ) { 
            (int256 paperAmount, int256 creditAmount) = IPerpetual(
                state.openPositions[trader][i]
            ).balanceOf(trader);
            Types.RiskParams storage params = state.perpRiskParams[
                state.openPositions[trader][i]
            ];
            int256 price = SafeCast.toInt256(
                IMarkPriceSource(params.markPriceSource).getMarkPrice()
            );

            netPositionValue += paperAmount.decimalMul(price) + creditAmount;
            uint256 exposureIncrement = paperAmount.decimalMul(price).abs();
            exposure += exposureIncrement;
            maintenanceMargin +=
                (exposureIncrement * params.liquidationThreshold) /
                Types.ONE;

            unchecked {
                ++i;
            }
        }
    }
```

So a malicious user can add enough positions so that the gas of liquidation transaction reaches the gas limit of a block and always revert.

## Vulnerability Detail

## Impact
Malicious user can prevent liquidation by openning lots of positions due to gas limit

## Code Snippet
https://github.com/JOJOexchange/smart-contract-EVM/blob/4a95a8e9a6367ae88dc827e29467229cb5bbad4f/contracts/lib/Liquidation.sol#L53-L85
## Tool used

Manual Review

## Recommendation
put a limit on total number of open positions for each user