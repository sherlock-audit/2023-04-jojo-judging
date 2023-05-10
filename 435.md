ArbitraryExecution

medium

# `newEmergencyOracle` can be called by anyone


## Summary

`newEmergencyOracle` can be called by anyone.

## Vulnerability Detail

The [`newEmergencyOracle` function](https://github.com/JOJOexchange/smart-contract-EVM/blob/4a95a8e9a6367ae88dc827e29467229cb5bbad4f/contracts/adaptor/emergencyOracle.sol#L54) in the `EmergencyOracleFactory` contract can be called by anyone. This means any address can create a new `emergencyOracle` contract from the `EmergencyOracleFactory` that was deployed by JOJO.

## Impact

This can cause confusion as to which `emergencyOracle` contract is valid, and which is fake. Each `emergencyOracle` contract's `price` is set by the owner, so if the owner is a malicious address that created a fake `emergencyOracle`, they will be able to manipulate the price of the oracle. If a fake oracle is accidentally set as an `emergencyOracle` in the JOJO protocol, this will result in price manipulation.

## Code Snippet

https://github.com/JOJOexchange/smart-contract-EVM/blob/4a95a8e9a6367ae88dc827e29467229cb5bbad4f/contracts/adaptor/emergencyOracle.sol#L54

```solidity
function newEmergencyOracle(string calldata description) external {
    address newOracle = address(
        new EmergencyOracle(msg.sender, description)
    );
    emit NewEmergencyOracle(msg.sender, newOracle);
}
```

## Tool used

Manual review.

## Recommendation

Have the [`EmergencyOracleFactory`](https://github.com/JOJOexchange/smart-contract-EVM/blob/4a95a8e9a6367ae88dc827e29467229cb5bbad4f/contracts/adaptor/emergencyOracle.sol#L51) contract inherit `Ownable`, and if necessary, add a `constructor` to allow the ownership of the `EmergencyOracleFactory` to be transferred to the desired protocol address.
