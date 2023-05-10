0x52

medium

# chainlinkAdaptor uses the same heartbeat for both feeds which is highly dangerous

## Summary
chainlinkAdaptor uses the same heartbeat for both feeds when checking if the data feed is fresh. The issue with this is that the [USDC/USD](https://data.chain.link/ethereum/mainnet/stablecoins/usdc-usd) oracle has a 24 hour heartbeat, whereas the [average](https://data.chain.link/ethereum/mainnet/crypto-usd/eth-usd) has a heartbeat of 1 hour. Since they use the same heartbeat the heartbeat needs to be slower of the two or else the contract would be nonfunctional most of the time. The issue is that it would allow the consumption of potentially very stale data from the non-USDC feed.

## Vulnerability Detail

See summary

## Impact

Either near constant downtime or insufficient staleness checks

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/adaptor/chainlinkAdaptor.sol#L43-L55

## Tool used

Manual Review

## Recommendation

Use two separate heartbeat periods