J4de

medium

# `JOJOOracleAdaptor.sol#getAssetPrice` uses a hardcoded decimal

## Summary

`JOJOOracleAdaptor.sol#getAssetPrice` uses a hardcoded decimal

## Vulnerability Detail

```solidity
File: oracle/JOJOOracleAdaptor.sol
 26     function getAssetPrice() external view override returns (uint256) {
 27         /*uint80 roundID*/
 28         (, int256 price,, uint256 updatedAt,) = IChainLinkAggregator(chainlink).latestRoundData();
 29         (, int256 USDCPrice,, uint256 USDCUpdatedAt,) = IChainLinkAggregator(USDCSource).latestRoundData()    ;
 30
 31         require(block.timestamp - updatedAt <= heartbeatInterval, "ORACLE_HEARTBEAT_FAILED");
 32         require(block.timestamp - USDCUpdatedAt <= heartbeatInterval, "USDC_ORACLE_HEARTBEAT_FAILED");
 33 >>      uint256 tokenPrice = (SafeCast.toUint256(price) * 1e8) / SafeCast.toUint256(USDCPrice);
 34         return tokenPrice * JOJOConstant.ONE / decimalsCorrection;
 35     }
```

The `getAssetPrice` function obtains the prices of Token and USDC from chainlink respectively, and divides them to get the price of Token/USDC. Chainlink usually provides USD or ETH trading pair prices, the decimal of USD trading pairs is 8, and the decimal of ETH trading pairs is 18. For example, the decimal of ETH/USD is 8, while the decimal of USD/ETH trading pair is 8.

If for a certain Token, chainlink only provides ETH transaction pair, then the `JOJOOracleAdaptor.sol` contract obviously can only use ETH trading pair. At this time, decimal should be 18 instead of 8, and line 33 uses `1e8` fixedly.

Same problem in `chainlinkAdaptor.sol` contract.

## Impact

May get unexpected price

## Code Snippet

https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/oracle/JOJOOracleAdaptor.sol#L26-L35

## Tool used

Manual Review

## Recommendation

It is recommended to dynamically get decimal

```diff
    function getAssetPrice() external view override returns (uint256) {
        /*uint80 roundID*/
        (, int256 price,, uint256 updatedAt,) = IChainLinkAggregator(chainlink).latestRoundData();
        (, int256 USDCPrice,, uint256 USDCUpdatedAt,) = IChainLinkAggregator(USDCSource).latestRoundData();

        require(block.timestamp - updatedAt <= heartbeatInterval, "ORACLE_HEARTBEAT_FAILED");
        require(block.timestamp - USDCUpdatedAt <= heartbeatInterval, "USDC_ORACLE_HEARTBEAT_FAILED");
-       uint256 tokenPrice = (SafeCast.toUint256(price) * 1e8) / SafeCast.toUint256(USDCPrice);
+       uint256 tokenPrice = (SafeCast.toUint256(price) * IChainLinkAggregator(chainlink).decimals()) / SafeCast.toUint256(USDCPrice);
        return tokenPrice * JOJOConstant.ONE / decimalsCorrection;
    }
```
