nobody2018

medium

# Subaccount#execute lacks payable

## Summary

`Subaccount#execute` lacks `payable`. If `value` in `Subaccount#execute` is not zero, it could always revert.

## Vulnerability Detail

`Subaccount#execute` lacks `payable`. The caller cannot send the value.

```solidity
function execute(address to, bytes calldata data, uint256 value) external onlyOwner returns (bytes memory){
        require(to != address(0));
->      (bool success, bytes memory returnData) = to.call{value: value}(data);
        if (!success) {
            assembly {
                let ptr := mload(0x40)
                let size := returndatasize()
                returndatacopy(ptr, 0, size)
                revert(ptr, size)
            }
        }
        emit ExecuteTransaction(owner, address(this), to, data, value);
        return returnData;
    }
```

The `Subaccount` contract does not implement `receive() payable` or `fallback() payable`, so it is unable to receive value (eth) . Therefore, `Subaccount#execute` needs to add `payable`.

## Impact

`Subaccount#execute` cannot work if `value` != 0.

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/subaccount/Subaccount.sol#L45-L58

## Tool used

Manual Review

## Recommendation

Add a `receive() external payable` to the contract or `execute()` to add a `payable` modifier.