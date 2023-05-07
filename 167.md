0xMojito

medium

# Operator should not be able to liquidate in JUSDBank

## Summary
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L143-L153

## Vulnerability Detail

The [JOJO Docs](https://jojo-docs.netlify.app/operator/#what-is-the-operator) states that:
> The operator is the client's agent and can sign an order on your behalf, but cannot withdraw or liquidate for the client.

However, the operator is allowed to liquidate in `JUSDBank.liquidate()` as shown in the code with the modifier `isValidOperator()`.

## Impact
This implementation is not consistent with the documentation. An operator could potentially behave maliciously and cause the liquidator to lose funds.

## Code Snippet
```solidity
function liquidate(
        address liquidated,
        address collateral,
        address liquidator,
        uint256 amount,
        bytes memory afterOperationParam,
        uint256 expectPrice
    )
        external
        override
        isValidOperator(msg.sender, liquidator) // @audit docs said operator cannot withdraw or liquidate
        nonFlashLoanReentrant
        returns (DataTypes.LiquidateData memory liquidateData)
    {
```

## Tool used

Manual Review

## Recommendation
Consider not allowing operator to liquidate in the codebase or updating the docs.
