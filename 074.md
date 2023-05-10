Ruhum

high

# JUSD borrow fee rate is less than it should be

## Summary
The borrow fee rate calculation is wrong causing the protocol to take less fees than it should.

## Vulnerability Detail
The borrowFeeRate is calculated through `getTRate()`:

```sol
    function getTRate() public view returns (uint256) {
        uint256 timeDifference = block.timestamp - uint256(lastUpdateTimestamp);
        return
            t0Rate +
            (borrowFeeRate * timeDifference) /
            JOJOConstant.SECONDS_PER_YEAR;
    }
```
`t0Rate` is initialized as `1e18` in the test contracts:

```sol
    constructor(
        uint256 _maxReservesNum,
        address _insurance,
        address _JUSD,
        address _JOJODealer,
        uint256 _maxPerAccountBorrowAmount,
        uint256 _maxTotalBorrowAmount,
        uint256 _borrowFeeRate,
        address _primaryAsset
    ) {
        // ...
        t0Rate = JOJOConstant.ONE;
    }
```

`SECONDS_PER_YEAR` is equal to `365 days` which is `60 * 60 * 24 * 365 = 31536000`:

```sol
library JOJOConstant {
    uint256 public constant SECONDS_PER_YEAR = 365 days;
}
```

As time passes, `getTRate()` value will increase. When a user borrows JUSD the contract doesn't save the actual amount of JUSD they borrow, `tAmount`. Instead, it saves the current "value" of it, `t0Amount`:

```sol
    function _borrow(
        DataTypes.UserInfo storage user,
        bool isDepositToJOJO,
        address to,
        uint256 tAmount,
        address from
    ) internal {
        uint256 tRate = getTRate();
        //        tAmount % tRate ？ tAmount / tRate + 1 ： tAmount % tRate
        uint256 t0Amount = tAmount.decimalRemainder(tRate)
            ? tAmount.decimalDiv(tRate)
            : tAmount.decimalDiv(tRate) + 1;
        user.t0BorrowBalance += t0Amount;
```

When you repay the JUSD, the same calculation is done again to decrease the borrowed amount. Meaning, as time passes, you have to repay more JUSD.

Let's say that JUSDBank was live for a year with a borrowing fee rate of 10% (1e17). `getTRate()` would then return:
$1e18 + 1e17 * 31536000 / 31536000 = 1.1e18$

If the user now borrows 1 JUSD we get: $1e6 * 1e18 / 1.1e18 ~= 909091$ for `t0Amount`. That's not the expected 10% decrease. Instead, it's about 9.1%.

## Impact
Users are able to borrow JUSD for cheaper than expected

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L274-L286
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/lib/JOJOConstant.sol#L7
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L37
## Tool used

Manual Review

## Recommendation
Change formula to:
`t0Amount = tAmount - tAmount.decimalMul(tRate)` where `t0Rate` is initialized with `0` instead of `1e18`.
