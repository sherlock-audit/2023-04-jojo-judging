Inspex

medium

# Unable to liquidate USDC blacklisted user's loan due to transferring leftover collateral back in USDC


## Summary
During the loan liquidation process, any remaining collateral will be swapped to USDC tokens and transferred to the liquidated user. However, if the USDC contract blacklists the liquidated user, the liquidation transaction will be revert. As a result, the user's loan will be unable to be liquidated if they have been blacklisted by the USDC token contract.


## Vulnerability Detail

During the liquidation process, any remaining tokens will be transferred to the owner of the loan. However, if the loan owner has been blacklisted by USDC token, this flow will be reverted due to the code shown below.

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L199-L204

As a result, users who have been blacklisted by USDC will be unable to liquidate their loan positions during the period of the blacklisting.

## Impact
The liquidation process might DoS due to its reliance on paying back remaining tokens in USDC only. This will error where transferring USDC tokens to blacklisted users can cause the transaction to be reverted, disrupting the liquidation flow. This will result in a bad debt for the platform.

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L199-L204

## Tool used

Manual Review

## Recommendation
We suggest implementing one or all of the following solutions:
1. Prevent USDC blacklisted users from opening a loan position until they are no longer blacklisted. This can be done by implementing a blacklist check during the borrowing process.
2. Remove the transfer of remaining USDC tokens to the liquidated user during the liquidation flow. Instead, allow the user to withdraw their remaining USDC tokens on their own after the liquidation process is complete.