0x52

medium

# JUSDBank users can bypass individual collateral borrow limits

## Summary

JUSDBank imposes individual borrow caps on each collateral. The issue is that this can be bypassed due to the fact that withdraw and borrow use different methods to determine if an account is safe. 

## Vulnerability Detail

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L105-L117

        function borrow(
            uint256 amount,
            address to,
            bool isDepositToJOJO
        ) external override nonReentrant nonFlashLoanReentrant{
            //     t0BorrowedAmount = borrowedAmount /  getT0Rate
            DataTypes.UserInfo storage user = userInfo[msg.sender];
            _borrow(user, isDepositToJOJO, to, amount, msg.sender);
            require(
                _isAccountSafeAfterBorrow(user, getTRate()),
                JUSDErrors.AFTER_BORROW_ACCOUNT_IS_NOT_SAFE
            );
        }

When borrowing the contract calls _isAccountSafeAfterBorrow. This imposes a max borrow on each collateral type that guarantees that the user cannot borrow more than the max for each collateral type. The issues is that withdraw doesn't impose this cap. This allows a user to bypass this cap as shown in the example below.

Example:
Assume WETH and WBTC both have a cap of 10,000 borrow. The user deposits $30,000 WETH and takes a flashloand for $30,000 WBTC. Now they deposit both and borrow 20,000 JUSD. They then withdraw all their WBTC to repay the flashloan and now they have borrowed 20,000 against $30000 in WETH

## Impact

Deposit caps can be easily surpassed creating systematic risk for the system

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L105-L117

## Tool used

Manual Review

## Recommendation

Always use _isAccountSafeAfterBorrow