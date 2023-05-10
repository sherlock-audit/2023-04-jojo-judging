0x52

medium

# Multiple contracts utilize swaps which can leave assets stranded in the contract

## Summary

A large number of helper contracts utilize swaps but never transfer the excess asset back to the user in the event that the swap leaves some assets left over

## Vulnerability Detail

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/stableCoin/DepositStableCoinToDealer.sol#L30-L49

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

        uint256 USDCAmount = IERC20(USDC).balanceOf(address(this));
        require(USDCAmount >= minReceive,"receive amount is too small");
        IERC20(USDC).approve(JOJODealer, USDCAmount);
        IDealer(JOJODealer).deposit(USDCAmount, 0, to);

The swap inside DepositStableCoinToDealer.sol is a perfect example of a swap that could leave assets stranded in the contract. We can see above that the assets are transferred into the contract and the swap happens but there is no check that the swap used all the assets. This allows portions of funds to be stranded in the contract where they can be stolen.

## Impact

Swaps may leave funds stranded

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/stableCoin/DepositStableCoinToDealer.sol#L23-L50

## Tool used

Manual Review

## Recommendation

The contract should check if there is a remaining balance after the swap and transfer it to the appropriate party