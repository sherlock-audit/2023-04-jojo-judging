Delvir0

high

# Insufficient signature data check could lead to signature replay

## Summary
Due to missing nonce check, a signature of a trade could be replayed by an attacker (grieving)
## Vulnerability Detail
Signature is checked by two aspects:
1. `require(recoverSigner == order.signer)`
2. `require(Trading._info2Expiration(order.info) >= block.timestamp`

Since there's no nonce, the signature is not unique and therefore can be replayed within the `_info2Expiration` time.
## Impact
Order can be placed multiple times due to a signature replay attack
## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/impl/JOJOExternal.sol#L139-L146
## Tool used

Manual Review

## Recommendation
Implement a nonce check