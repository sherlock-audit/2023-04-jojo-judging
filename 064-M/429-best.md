GimelSec

medium

# `Funding.requestWithdraw` should add an expiry time. Or the `withdrawTimeLock` can be easily bypassed.

## Summary

In `Funding.requestWithdraw`, it would set the timelock. And the user can only call `executeWithdraw` after the pending time is over. However, a user can easily bypass the timelock.

## Vulnerability Detail

`Funding.requestWithdraw` sets a timelock at state.withdrawExecutionTimestamp.
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/lib/Funding.sol#L84
```solidity
    function requestWithdraw(
        Types.State storage state,
        uint256 primaryAmount,
        uint256 secondaryAmount
    ) external {
        state.pendingPrimaryWithdraw[msg.sender] = primaryAmount;
        state.pendingSecondaryWithdraw[msg.sender] = secondaryAmount;
        state.withdrawExecutionTimestamp[msg.sender] =
            block.timestamp +
            state.withdrawTimeLock;
        emit RequestWithdraw(
            msg.sender,
            primaryAmount,
            secondaryAmount,
            state.withdrawExecutionTimestamp[msg.sender]
        );
    }
```

And the timelock is checked in `Funding.executeWithdraw()`
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/lib/Funding.sol#L102
```solidity
    function executeWithdraw(
        Types.State storage state,
        address to,
        bool isInternal
    ) external {
        require(
            state.withdrawExecutionTimestamp[msg.sender] <= block.timestamp,
            Errors.WITHDRAW_PENDING
        );
        …
    }
```

However, there is an easy method to bypass the timelock. Since there is no check on the amount to withdraw. A user can call `requestWithdraw` before depositing any funds. And when the user wants to withdraw the fund, the user can directly call `executeWithdraw` since the timelock has already passed.

## Impact

`state.withdrawTimeLock` can be easily bypassed.

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/lib/Funding.sol#L84
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/lib/Funding.sol#L102
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/impl/JOJOExternal.sol#L35
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/impl/JOJOExternal.sol#L43


## Tool used

Manual Review

## Recommendation

Add an expiry check
```solidity
    function executeWithdraw(
        Types.State storage state,
        address to,
        bool isInternal
    ) external {
        require(
            state.withdrawExecutionTimestamp[msg.sender] <= block.timestamp,
            Errors.WITHDRAW_PENDING
        );
        require(
            state.withdrawExecutionTimestamp[msg.sender] + state.withdrawExpiredTime >= block.timestamp,
            Errors.WITHDRAW_EXPIRED
        );
        …
    }
```
