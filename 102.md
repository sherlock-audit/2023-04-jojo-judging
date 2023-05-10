bytes032

medium

# Missing checks for whether Arbitrum Sequencer is active



## Summary

Missing checks for whether Arbitrum Sequencer is active

## Vulnerability Detail

the onchain deployment context is changed, in prev contest the protocol only attemps to deploy the code to ethereum while in the current contest

the protocol intends to deploy to arbtrium as well!

Chainlink recommends that users using price oracles, check whether the Arbitrum sequencer is active

[https://docs.chain.link/data-feeds#l2-sequencer-uptime-feeds](https://docs.chain.link/data-feeds#l2-sequencer-uptime-feeds)

If the sequencer goes down, the index oracles may have stale prices, since L2-submitted transactions (i.e. by the aggregating oracles) will not be processed.

## Impact

Stale prices, e.g. if USDC were to de-peg while the sequencer is offline, stale price is used and can result in false liquidation or over-borrowing.

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/oracle/JOJOOracleAdaptor.sol#L26-L35

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

## Tool used

Manual Review

## Recommendation

Use sequencer oracle to determine whether the sequencer is offline or not, and don't allow orders to be executed while the sequencer is offline.