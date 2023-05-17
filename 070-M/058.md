rvierdiiev

medium

# JUSDOperation._addReserve allows to add more reserves than maxReservesNum and readd same reserve

## Summary
JUSDOperation._addReserve allows to add more reserves than maxReservesNum, which creates management risk for the team. Also it allows to readd same reserve again, which will corrupt `reservesNum` variable
## Vulnerability Detail
JUSDOperation._addReserve allows to add new reserve. It is restricted by `maxReservesNum` variable.
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDOperation.sol#L94-L101
```solidity
    function _addReserve(address collateral) private {
        require(
            reservesNum <= maxReservesNum,
            JUSDErrors.NO_MORE_RESERVE_ALLOWED
        );
        reservesList.push(collateral);
        reservesNum += 1;
    }
```

The problem is that `reservesNum` is increased only at the end of function, which allows to create more reserves than `maxReservesNum` amount. You can add `maxReservesNum + 1` reserves.

Another problem that i would like to describe in this bug is that `initReserve` function doesn't check [if reserve is already added](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDOperation.sol#L59-L92). This allows to overwrite already existing reserve, but [`reservesNum` will be incorrectly increased](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDOperation.sol#L100) in this case, which will corrupt this variable.
## Impact
Creates management risks for the protocol.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
1. Check that reserve was not added before.
2. When create new reserve do check like this
```solidity
require(
            reservesNum < maxReservesNum,
            JUSDErrors.NO_MORE_RESERVE_ALLOWED
        );
```