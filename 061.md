rvierdiiev

medium

# JUSDOperation.updateBorrowFeeRate doesn't give time to react for users

## Summary
JUSDOperation.updateBorrowFeeRate allows owner to change borrow rate to any new rate. This is dangerous for protocol users as they don't have enough time to react, so their position can become liquidatable.
## Vulnerability Detail
`JUSDOperation.updateBorrowFeeRate` JUSDOperation.updateBorrowFeeRate allows owner to change borrow rate to any new rate.
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDOperation.sol#L168-L173
```solidity
    function updateBorrowFeeRate(uint256 _borrowFeeRate) external onlyOwner {
        t0Rate = getTRate();
        lastUpdateTimestamp = uint32(block.timestamp);
        borrowFeeRate = _borrowFeeRate;
        emit UpdateBorrowFeeRate(_borrowFeeRate, t0Rate, lastUpdateTimestamp);
    }
```
Once it's changed, then new rate is used to calculate borrowing fees starting form this time.

This is dangerous for the protocol users, as some positions can become liquidatable soon after fee changing, so they will loose some funds.
## Impact
Positions can become liquidatable after fee changing which causes loose of funds for position owners.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Use some time lock, so users can deposit additional collateral.