volodya

medium

# ChainlinkExpandAdaptor and JOJOOracleAdaptor will return the wrong price for asset if underlying aggregator hits minAnswer

## Summary
ChainlinkExpandAdaptor and JOJOOracleAdaptor will return the wrong price for asset if underlying aggregator hits minAnswer
## Vulnerability Detail
Chainlink aggregators have a built in circuit breaker if the price of an asset goes outside of a predetermined price band. The result is that if an asset experiences a huge drop in value (i.e. LUNA crash) the price of the oracle will continue to return the minPrice instead of the actual price of the asset. This would allow user to continue borrowing with the asset but at the wrong price. This is exactly what happened to [Venus on BSC when LUNA imploded](https://rekt.news/venus-blizz-rekt/).

```solidity
    function getAssetPrice() external view override returns (uint256) {
        /*uint80 roundID*/
        (, int256 price,, uint256 updatedAt,) = IChainLinkAggregator(chainlink).latestRoundData();
        (, int256 USDCPrice,, uint256 USDCUpdatedAt,) = IChainLinkAggregator(USDCSource).latestRoundData();

        require(block.timestamp - updatedAt <= heartbeatInterval, "ORACLE_HEARTBEAT_FAILED");
        require(block.timestamp - USDCUpdatedAt <= heartbeatInterval, "USDC_ORACLE_HEARTBEAT_FAILED");
        uint256 tokenPrice = (SafeCast.toUint256(price) * 1e8) / SafeCast.toUint256(USDCPrice);
        return tokenPrice * JOJOConstant.ONE / decimalsCorrection;
    }

```
[src/oracle/JOJOOracleAdaptor.sol#L26](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/oracle/JOJOOracleAdaptor.sol#L26)
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
[contracts/adaptor/chainlinkAdaptor.sol#L43](https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/adaptor/chainlinkAdaptor.sol#L43)
## Impact
In the event that an asset crashes (i.e. LUNA) the protocol can be manipulated to give out loans at an inflated price

## Code Snippet

## Tool used
Related issue https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/18
Manual Review

## Recommendation
should check the returned answer against the minPrice/maxPrice and revert if the answer is outside of the bounds:
Do the same for ChainlinkExpandAdaptor
```diff
/*
    Copyright 2022 JOJO Exchange
    SPDX-License-Identifier: BUSL-1.1*/

pragma solidity 0.8.9;

import "../Interface/IPriceChainLink.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import { IChainLinkAggregator } from "../Interface/IChainLinkAggregator.sol";
import "@openzeppelin/contracts/utils/math/SafeCast.sol";
import "../lib/JOJOConstant.sol";

contract JOJOOracleAdaptor is IPriceChainLink, Ownable {
    address public immutable chainlink;
    uint256 public immutable decimalsCorrection;
    uint256 public immutable heartbeatInterval;
    address public immutable USDCSource;

+    //hardcoded price bounds used by chainlink
+    int192 immutable maxPrice;
+    int192 immutable minPrice;
+    int192 immutable maxPriceUSDC;
+    int192 immutable minPriceUSDC;

    constructor(address _source, uint256 _decimalCorrection, uint256 _heartbeatInterval, address _USDCSource) {
        chainlink = _source;
        decimalsCorrection = 10 ** _decimalCorrection;
        heartbeatInterval = _heartbeatInterval;
        USDCSource = _USDCSource;

+       minPrice = IChainLinkAggregator(chainlink).minAnswer();
+       maxPrice = IChainLinkAggregator(chainlink).maxAnswer();
+       minPriceUSDC = IChainLinkAggregator(USDCSource).minAnswer();
+       maxPriceUSDC = IChainLinkAggregator(USDCSource).maxAnswer();
    }

    function getAssetPrice() external view override returns (uint256) {
        /*uint80 roundID*/
        (, int256 price,, uint256 updatedAt,) = IChainLinkAggregator(chainlink).latestRoundData();
        (, int256 USDCPrice,, uint256 USDCUpdatedAt,) = IChainLinkAggregator(USDCSource).latestRoundData();

+        require(price < maxPrice, "Upper price bound breached");
+        require(price > minPrice, "Lower price bound breached");
+        require(USDCPrice < maxPriceUSDC, "Upper price bound breached");
+        require(USDCPrice > minPriceUSDC, "Lower price bound breached");

        require(block.timestamp - updatedAt <= heartbeatInterval, "ORACLE_HEARTBEAT_FAILED");
        require(block.timestamp - USDCUpdatedAt <= heartbeatInterval, "USDC_ORACLE_HEARTBEAT_FAILED");
        uint256 tokenPrice = (SafeCast.toUint256(price) * 1e8) / SafeCast.toUint256(USDCPrice);
        return tokenPrice * JOJOConstant.ONE / decimalsCorrection;
    }
}

```
