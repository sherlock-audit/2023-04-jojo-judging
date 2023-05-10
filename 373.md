cccz

medium

# FlashLoanLiquidate.JOJOFlashLoan has no slippage control when swapping USDC

## Summary
FlashLoanLiquidate.JOJOFlashLoan has no slippage control when swapping USDC
## Vulnerability Detail
In both GeneralRepay.repayJUSD and FlashLoanRepay.JOJOFlashLoan, the user-supplied minReceive parameter is used for slippage control when swapping USDC. 
```solidity
    function JOJOFlashLoan(
        address asset,
        uint256 amount,
        address to,
        bytes calldata param
    ) external {
        (address approveTarget, address swapTarget, uint256 minReceive, bytes memory data) = abi
            .decode(param, (address, address, uint256, bytes));
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
        require(USDCAmount >= minReceive, "receive amount is too small");
...
    function repayJUSD(
        address asset,
        uint256 amount,
        address to,
        bytes memory param
    ) external {
        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
        uint256 minReceive;
        if (asset != USDC) {
            (address approveTarget, address swapTarget, uint256 minAmount, bytes memory data) = abi
                .decode(param, (address, address, uint256, bytes));
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
            minReceive = minAmount;
        }

        uint256 USDCAmount = IERC20(USDC).balanceOf(address(this));
        require(USDCAmount >= minReceive, "receive amount is too small");
```
However, this is not done in FlashLoanLiquidate.JOJOFlashLoan, and the lack of slippage control may expose the user to sandwich attacks when swapping USDC.
## Impact
The lack of slippage control may expose the user to sandwich attacks when swapping USDC.
## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/flashloanImpl/FlashLoanLiquidate.sol#L46-L78

## Tool used

Manual Review

## Recommendation
Consider making FlashLoanLiquidate.JOJOFlashLoan use the minReceive parameter for slippage control when swapping USDC.