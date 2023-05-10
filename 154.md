rvierdiiev

medium

# FundingRateUpdateLimiter.updateFundingRate will allow very big change when first time applied to the perp

## Summary
FundingRateUpdateLimiter.updateFundingRate will allow very big change when first time applied to the perp.
## Vulnerability Detail
`FundingRateUpdateLimiter.updateFundingRate` allows keeper to provide funding rate for the perpetual contracts. 
The purpose of `FundingRateUpdateLimiter` contract is to disallow big changes of funding rate for the perpetual contract.
It should allow [maximum `speedMultiplier*liquidationThreshold` change per day](https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/fundingRateKeeper/FundingRateUpdateLimiter.sol#L59).

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/fundingRateKeeper/FundingRateUpdateLimiter.sol#L37-L71
```solidity
    function updateFundingRate(
        address[] calldata perpList,
        int256[] calldata rateList
    ) external onlyOwner {
        for (uint256 i = 0; i < perpList.length;) {
            address perp = perpList[i];
            int256 oldRate = IPerpetual(perp).getFundingRate();
            uint256 maxChange = getMaxChange(perp);
            require(
                (rateList[i] - oldRate).abs() <= maxChange,
                "FUNDING_RATE_CHANGE_TOO_MUCH"
            );
            fundingRateUpdateTimestamp[perp] = block.timestamp;
            unchecked {
                ++i;
            }
        }


        IDealer(dealer).updateFundingRate(perpList, rateList);
    }


    // limit funding rate change speed
    // can not exceed speedMultiplier*liquidationThreshold
    function getMaxChange(address perp) public view returns (uint256) {
        Types.RiskParams memory params = IDealer(dealer).getRiskParams(perp);
        uint256 markPrice = IMarkPriceSource(params.markPriceSource)
            .getMarkPrice();
        uint256 timeInterval = block.timestamp -
            fundingRateUpdateTimestamp[perp];
        uint256 maxChangeRate = (speedMultiplier *
            timeInterval *
            params.liquidationThreshold) / (1 days);
        uint256 maxChange = (maxChangeRate * markPrice) / Types.ONE;
        return maxChange;
    }
```
For each perpetual contract function stores `fundingRateUpdateTimestamp` variable. This is needed to check how many time has passed since last funding rate update for the perpetual.
This timestamp will be used inside `getMaxChange` function, to make 1 day maximum change control.

The problem is that `fundingRateUpdateTimestamp` will not contain any value for the perpetual, when `updateFundingRate` will be called for first time for it.
Because of that, `maxChange` calculation will use `block.timestamp` as `timeInterval`. So it will allow to change rate up to `block.timestamp / 1 days * speedMultiplier * params.liquidationThreshold` which is very big number.
## Impact
Funding rate can be set to any value when is provided first time for the perpetual.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Need somehow handle first time call for perpetual to restrict big changes.