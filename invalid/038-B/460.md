ArbitraryExecution

high

# `updateBorrowFeeRate` may erase the interest accrual from the previous `lastUpdateTimestamp` to the current `block.timestamp`

## Summary

`updateBorrowFeeRate` may erase the interest accrual from the previous `lastUpdateTimestamp` to the current `block.timestamp` for borrowers that have not borrowed or repaid JUSD from the `JUSDBank` contract since their first borrow.

## Vulnerability Detail

The `JUSDBank` contract only stores accrued interest on a user's borrow amount is when a user calls [`_borrow`](https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDBank.sol#L286) and [`_repay`](https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDBank.sol#L326). Otherwise, the `tRate` is calculated for a given borrower and used as part of calling [`_isAccountSafe`](https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDView.sol#L123) to determine if a user's account is still healthy with the interest accrued; this does not save the interest accrued to the borrower's state, however.

Because JOJO does not checkpoint the interest rate accrual of borrowers, when [`updateBorrowFeeRate`](https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDOperation.sol#L168) is called and the `lastUpdateTimestamp` is reset, all the previously accrued interest will be lost since the last time a borrower has either borrowed more JUSD or repaid a portion of their JUSD borrow. This is due to the [calculation](https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDBankStorage.sol#L52) in [`getTRate`](https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDBankStorage.sol#L51) which uses the `lastUpdateTimestamp` of the borrow rate to determine the interest accrued, instead of storing a timestamp for each individual borrower.

## Impact

If a user borrows JUSD, and then doesn't repay or borrow more JUSD with the protocol before `updateBorrowFeeRate` is called, all the interest that the borrower should have to pay will be erased. A user could then repay their borrow without having paid any interest.

## Code Snippet

https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDOperation.sol#L168-L173

```solidity
function updateBorrowFeeRate(uint256 _borrowFeeRate) external onlyOwner {
    t0Rate = getTRate();
    lastUpdateTimestamp = uint32(block.timestamp);
    borrowFeeRate = _borrowFeeRate;
    emit UpdateBorrowFeeRate(_borrowFeeRate, t0Rate, lastUpdateTimestamp);
}
```

https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDBankStorage.sol#L51-L57

```solidity
function getTRate() public view returns (uint256) {
    uint256 timeDifference = block.timestamp - uint256(lastUpdateTimestamp);
    return
        t0Rate +
        (borrowFeeRate * timeDifference) /
        JOJOConstant.SECONDS_PER_YEAR;
}
```

## Tool used

Manual review.

## Recommendation

Consider accruing interest for a borrower each time they interact with the `JUSDBank` contract. Additionally, JOJO should consider removing the ability to change the borrow fee rate, as even if they did checkpoint the last time a user's timestamp was updated changing the `borrowFeeRate` can still cause incorrect behavior. This is because a user will then accrue interest at the new, different `borrowFeeRate` than whatever the previous rate was over the period of time of the previous fee rate, which would also be incorrect. If JOJO is concerned about the interest rate relative the total amount of JUSD borrowed, they should consider using a non-fixed borrowing rate equation that is proportional to the total amount of JUSD borrowed in `JUSDBank`.
