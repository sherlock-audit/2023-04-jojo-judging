deadrxsezzz

medium

# Uniswap getting the price from all available pools for certain token pair possesses a risk

## Summary
UniswapV3 oracle price can be easily manipulated

## Vulnerability Detail
Currently, the implementation of the UniswapV3 oracle gets the price of a token pair based on all available pools with said token pair.
```solidity
(uint256 uniswapPriceFeed,) = IStaticOracle(UNISWAP_V3_ORACLE).quoteAllAvailablePoolsWithTimePeriod(uint128(10**decimal), baseToken, quoteToken, period);
```
Uniswap allows for numerous pools to exist for the same token pair as long as they have different fee levels. For certain pairs (e.g. pools with 1% fee for stablecoins) some pools may exist, but have very little liquidity. This can make price manipulation very cheap , although TWAP is used.
An attacker could also deploy a pool for a certain fee level (to manipulate its price) when it does not exist already. 


## Impact
Uniswap oracle will return false data. A malicious user might manipulate Uniswap's oracle to return token's value to be higher than what it really is, allowing himself to profit off the protocol

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/adaptor/uniswapPriceAdaptor.sol#L49


## Tool used

Manual Review

## Recommendation
Either use specific pools to calculate the price of a token pair or add a requirement for liquidity pools to not have more than 50% difference in liquidity between eachother 

