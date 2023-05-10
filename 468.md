0x52

medium

# uniswapPriceAdaptor will function incorrectly if quote token isn't 18 dp

## Summary

The precision of the uniswapPriceAdaptor matches the decimals of the quote token which will cause the price returned to be incorrect

## Vulnerability Detail

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/oracle/UniswapPriceAdaptor.sol#L48-L55

    function getAssetPrice() external view returns (uint256) {
        (uint256 uniswapPriceFeed,) = IStaticOracle(UNISWAP_V3_ORACLE).quoteAllAvailablePoolsWithTimePeriod(uint128(10**decimal), baseToken, quoteToken, period);
        uint256 JOJOPriceFeed = EmergencyOracle(priceFeedOracle).getMarkPrice();
        uint256 diff = JOJOPriceFeed >= uniswapPriceFeed ? JOJOPriceFeed - uniswapPriceFeed : uniswapPriceFeed - JOJOPriceFeed;
        //JOJOPriceFeed(1 - impact) <= uniswapPriceFeed <= JOJOPriceFeed(1 + impact)
        require(diff * 1e18 / JOJOPriceFeed <= impact, "deviation is too big");
        return uniswapPriceFeed;
    }

Above we see that uniswapPriceAdaptor uses the quoteAllAvailablePoolsWithTimePeriod function.


https://github.com/Mean-Finance/uniswap-v3-oracle/blob/6888b16a6eefb82226b2086ed6d42f8bf4e10b69/solidity/contracts/StaticOracle.sol#L53-L61
  
    function quoteAllAvailablePoolsWithTimePeriod(
      uint128 _baseAmount,
      address _baseToken,
      address _quoteToken,
      uint32 _period
    ) external view override returns (uint256 _quoteAmount, address[] memory _queriedPools) {
      _queriedPools = _getQueryablePoolsForTiers(_baseToken, _quoteToken, _period);
      _quoteAmount = _quote(_baseAmount, _baseToken, _quoteToken, _queriedPools, _period);
    }

https://github.com/Mean-Finance/uniswap-v3-oracle/blob/6888b16a6eefb82226b2086ed6d42f8bf4e10b69/solidity/contracts/StaticOracle.sol#L158-L174

    function _quote(
      uint128 _baseAmount,
      address _baseToken,
      address _quoteToken,
      address[] memory _pools,
      uint32 _period
    ) internal view returns (uint256 _quoteAmount) {
      require(_pools.length > 0, 'No defined pools');
      OracleLibrary.WeightedTickData[] memory _tickData = new OracleLibrary.WeightedTickData[](_pools.length);
      for (uint256 i; i < _pools.length; i++) {
        (_tickData[i].tick, _tickData[i].weight) = _period > 0
          ? OracleLibrary.consult(_pools[i], _period)
          : OracleLibrary.getBlockStartingTickAndLiquidity(_pools[i]);
      }
      int24 _weightedTick = _tickData.length == 1 ? _tickData[0].tick : OracleLibrary.getWeightedArithmeticMeanTick(_tickData);
      return OracleLibrary.getQuoteAtTick(_weightedTick, _baseAmount, _baseToken, _quoteToken); <- @audit-issue returns raw tick
    }

Above we can see that the Uniswap oracle used returns the raw tick for the price. This means that it will return the price of the base token to the same precision as the quote token. Since USDC seems to be the quote token of choice and is 6 dp, this is very problematic, since it is expected throughout the code that the oracle is 18 dp in precision.

## Impact

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/oracle/UniswapPriceAdaptor.sol#L48-L55

## Tool used

Manual Review

## Recommendation

Adjust uniswapPriceFeed by the decimals of the quote token so that it is 18 dp