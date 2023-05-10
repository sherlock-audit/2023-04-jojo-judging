0x52

high

# All allowances to DepositStableCoinToDealer and GeneralRepay can be stolen due to unsafe call

## Summary

DepositStableCoinToDealer.sol and GeneralRepay.sol are helper contracts that allow a user to swap and enter JOJODealer and JUSDBank respectively. The issue is that the call is unsafe allowing the contract to call the token contracts directly and transfer tokens from anyone who has approved the contract.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/stableCoin/DepositStableCoinToDealer.sol#L30-L44

        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
        (address approveTarget, address swapTarget, bytes memory data) = abi
        .decode(param, (address, address, bytes));
        // if usdt
        IERC20(asset).approve(approveTarget, 0);
        IERC20(asset).approve(approveTarget, amount);
        (bool success, ) = swapTarget.call(data);
        if (success == false) {
            assembly {
                let ptr := mload(0x40)
                let size := returndatasize()
                returndatacopy(ptr, 0, size)
                revert(ptr, size)
            }
        }

We can see above that the call is totally unprotected allowing a user to make any call to any contract. This can be abused by calling the token contract and using the allowances of others. The attack would go as follows:

1. User A approves the contract for 100 USDT
2. User B sees this approval and calls depositStableCoin with the swap target as the USDT contract with themselves as the receiver
3. This transfers all of user A USDT to them

## Impact

All allowances can be stolen

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/stableCoin/DepositStableCoinToDealer.sol#L23C14-L50

## Tool used

Manual Review

## Recommendation

Only allow users to call certain whitelisted contracts.