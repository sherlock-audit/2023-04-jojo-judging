Bauer

high

# The price `uniswapPriceFeed` may be inaccurate.

## Summary
The protocol did not call `prepareAllAvailablePoolsWithTimePeriod()` function  to obtain the latest available Uniswap v3 pools before calling the `quoteAllAvailablePoolsWithTimePeriod()` function, which could result in the retrieved price being inaccurate.

## Vulnerability Detail
StaticOracle is a tool developed under Uniswap's grant program that aims to help developers integrate easily and fast with Uniswap's v3 TWAP oracles.
StaticOracle will allow developers to:
Prepare a set of pools (for example: by a token pair and a fee tier) to support a certain period of time (for example: 60 seconds).
Quote a TWAP for a set of pools (for example: by a token pair and a fee tier, or addresses).
The `UniswapPriceAdaptor.getMarkPrice()`  is used to retrieve the Uniswap v3 price feed for a specified base and quote token pair over a specified time period, and to verify that the JOJO price feed is within an allowed deviation from the Uniswap v3 price feed.
However, before calling the `IStaticOracle(UNISWAP_V3_ORACLE).quoteAllAvailablePoolsWithTimePeriod()` function to retrieve the quote amount of the quote token for the specified base amount over the specified time period, the protocol did not call the `prepareAllAvailablePoolsWithTimePeriod()` function to prepare all available Uniswap v3 pools for the specified pair of tokens over the specified time period. This could result in the "uniswapPriceFeed" price being inaccurate.
```solidity
    function getMarkPrice() external view returns (uint256) {
        (uint256 uniswapPriceFeed,) = IStaticOracle(UNISWAP_V3_ORACLE).quoteAllAvailablePoolsWithTimePeriod(uint128(10**decimal), baseToken, quoteToken, period);
        uint256 JOJOPriceFeed = EmergencyOracle(priceFeedOracle).getMarkPrice();
        uint256 diff = JOJOPriceFeed >= uniswapPriceFeed ? JOJOPriceFeed - uniswapPriceFeed : uniswapPriceFeed - JOJOPriceFeed;
        require(diff * 1e18 / JOJOPriceFeed <= impact, "deviation is too big");
        return uniswapPriceFeed;
    }

```

## Impact
The price `uniswapPriceFeed` may be inaccurate.

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/adaptor/uniswapPriceAdaptor.sol#L49
https://etherscan.io/address/0xB210CE856631EeEB767eFa666EC7C1C57738d438#code#F1#L88
## Tool used

Manual Review

## Recommendation
Call the `prepareAllAvailablePoolsWithTimePeriod()` function before calling the `quoteAllAvailablePoolsWithTimePeriod()`  function to prepare all available Uniswap v3 pools for the specified pair of tokens over the specified time period.
