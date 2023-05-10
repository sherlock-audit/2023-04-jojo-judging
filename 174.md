Bauer

medium

# Missing check of oracle's return price

## Summary
Missing check the parameter rawPrice return from chainlink oracle


## Vulnerability Detail
The lastRoundData()'s parameters according to [Chainlink](https://docs.chain.link/docs/data-feeds/price-feeds/api-reference/) are the following:
```solidity
function latestRoundData() external view
    returns (
        uint80 roundId,             //  The round ID.
        int256 answer,              //  The price.
        uint256 startedAt,          //  Timestamp of when the round started.
        uint256 updatedAt,          //  Timestamp of when the round was updated.
        uint80 answeredInRound      //  The round ID of the round in which the answer was computed.
    )
```
Function ChainlinkExpandAdaptor.getMarkPrice() has 1 checks with 2 parameters price, updatedAt but forget to check the price return value which lead to the 0 price data from chainlink.
```solidity
function getMarkPrice() external view returns (uint256 price) {
        int256 rawPrice;
        uint256 updatedAt;
        (, rawPrice, , updatedAt, ) = IChainlink(chainlink).latestRoundData();
        (, int256 USDCPrice,, uint256 USDCUpdatedAt,) = IChainlink(USDCSource).latestRoundData();
        require(
            block.timestamp - updatedAt <= heartbeatInterval,
            "ORACLE_HEARTBEAT_FAILED"
        );
        require(block.timestamp - USDCUpdatedAt <= heartbeatInterval, "USDC_ORACLE_HEARTBEAT_FAILED");
        uint256 tokenPrice = (SafeCast.toUint256(rawPrice) * 1e8) / SafeCast.toUint256(USDCPrice);
        return tokenPrice * 1e18 / decimalsCorrection;
    }
```

## Impact
Incorrect return price value lead to incorrect de-peg events trigger.


## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/adaptor/chainlinkAdaptor.sol#L43-L55

## Tool used

Manual Review

## Recommendation
Modify the function to have a check like this:
```solidity
(uint80 roundID, int256 rawPrice, , uint256 updatedAt, uint80 answeredInRound) = feed.latestRoundData();
require(rawPrice> 0, "Chainlink price <= 0"); 
require(answeredInRound >= roundID, "Stale price");
 require(
            block.timestamp - updatedAt <= heartbeatInterval,
            "ORACLE_HEARTBEAT_FAILED"
        );
```