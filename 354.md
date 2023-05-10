GalloDaSballo

medium

# Function UpdatePrimaryAsset can break the protocol and has no use

## Summary

Due to hardcoded Decimal Math, the Primary Asset should not be changed under any circumstance

## Vulnerability Detail

Changing `primaryAsset` will cause a multitude of issues such as:
- Incorrect accounting
- Liquidation of users
- Improper math for liquidation and operations

## Impact

Changing `primaryAsset` will cause issues which should be avoided, by either limiting the function to a one off usage or by removing it altogether

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDOperation.sol#L161-L173

```solidity
    function updatePrimaryAsset(address newPrimary) external onlyOwner {
        emit UpdatePrimaryAsset(primaryAsset, newPrimary);
        primaryAsset = newPrimary;
    }

    /// @notice update the borrow fee rate
    // t0Rate and lastUpdateTimestamp will be updated according to the borrow fee rate
    function updateBorrowFeeRate(uint256 _borrowFeeRate) external onlyOwner {
        t0Rate = getTRate();
        lastUpdateTimestamp = uint32(block.timestamp);
        borrowFeeRate = _borrowFeeRate;
        emit UpdateBorrowFeeRate(_borrowFeeRate, t0Rate, lastUpdateTimestamp);
    }
```

UpdatePrimary Asset breaks all accounting, no safe usage of the function

## Tool used

Manual Review

## Recommendation

If this is done to set the system up, ensure that the call is done from address(0) (unset value) to a set value

To migrate in the future, a re-deployment will always be necessary as to avoid liquidating laggards users
