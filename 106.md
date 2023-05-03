holyhansss

medium

# ChainlinkAdapterOracle uses potentially problematic BTC/USD Chainlink oracle to price WBTC.

## Summary
## Vulnerability Detail
When I asked a question about Oracle via DM, I received a response to refer to the address located in the deployment/ folder. Upon investigation, I found that the BTC/USD pair is being used, which I was able to confirm on [[ArbiScan](https://goerli.arbiscan.io/address/0x6550bc2301936011c1334555e62a87705a81c12c#readContract)](https://goerli.arbiscan.io/address/0x6550bc2301936011c1334555e62a87705a81c12c#readContract).

The Chainlink BTC/USD oracle is utilized to determine the price of WBTC. WBTC is essentially an asset that has been bridged, and if this bridge is compromised or fails, WBTC's value will no longer be equivalent to BTC. This may result in significant borrowing against an asset that is essentially worthless. Since the protocol still evaluates the asset via BTC/USD, the protocol will be faced not only with the bad debts caused by outstanding loans, but will also continue to issue bad loans, thereby increasing the amount of bad debt even further.

Here is [[report 0x52 wrote](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/9)](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/9) for same issue.

0x52 - [[ChainlinkAdapterOracle use BTC/USD chainlink oracle to price WBTC which is problematic if WBTC depegs](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/9)](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/9)

## Impact
If the WBTC bridge is compromised and WBTC depegs, the protocol will have a significant amount of bad debt.

## Code Snippet
https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/oracle/JOJOOracleAdaptor.sol#L26-L36

## Tool used

Manual Review

## Recommendation
I recommand to use WBTC/USD pair, not BTC/USD, or using a dual oracle system that includes both the Chainlink oracle and an on-chain liquidity-based oracle like UniV3 TWAP. If the price reported by the on-chain liquidity oracle falls below a certain threshold compared to the Chainlink oracle (e.g., 2% lower), borrowing should be stopped right away. By combining both oracles, the Chainlink oracle will prevent price manipulation, while the liquidity oracle will protect against depegging of the asset.