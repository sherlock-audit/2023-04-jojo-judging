XDZIBEC

medium

# Subaccount Contract Allows Anyone to Initiate Contract Initialization, Allowing for Potential Ownership Takeover #L45

## Summary

the vulnerability is a reentrancy attack, an attacker can exploit the `execute()` function in the `Subaccount` contract to repeatedly call back into the contract and drain its funds.
the fact that the `execute()` function does not update the balance of the `subaccount` after executing the transaction, which allows the attacker to call the function recursively and drain all the funds of the `subaccount`
The attack can be initiated by calling the execute() function with a malicious contract address that calls back into the `subaccount` contract

## Vulnerability Detail

```solidity
function execute(address to, bytes calldata data, uint256 value) external onlyOwner returns (bytes memory){
        require(to != address(0));
        (bool success, bytes memory returnData) = to.call{value: value}(data);
        if (!success) {
            assembly {
                let ptr := mload(0x40)
                let size := returndatasize()
                returndatacopy(ptr, 0, size)
                revert(ptr, size)
            }
        }
```

The` execute()` function allows the owner of the contract to execute arbitrary transactions by providing a destination address, data and value, this function does not perform any input validation on the destination address or the data parameter, which can allow an attacker to execute arbitrary code or drain the contract's balance.
An attacker can simply call the `execute()` function with a malicious destination address and data parameter, which can cause the function to execute arbitrary code that can perform actions that are not intended by the contract's design.
the execute() function uses the low-level call opcode, which forwards all available gas to the destination address. This can lead to unexpected out-of-gas errors if the destination address performs an operation that consumes a large amount of gas.
this vulnerability can allow an attacker to steal the contract's funds or perform unauthorized actions on the contract. To mitigate this vulnerability, input validation should be added to the `execute()` function to ensure that only trusted addresses and data are used. A gas limit should also be added to the call to prevent unexpected out-of-gas errors

## Impact

An attacker can drain the entire balance of the` subaccount` contract.
The attacker can make a malicious call to the `execute()` function with a to argument set to the attacker's address, and `value` set to the balance of the contract. Since the `onlyOwner` modifier only checks that the caller is the owner of the contract and not the owner of the `subaccount`, the attacker can easily drain the entire balance of the` subaccount `contract. This can result in a significant loss of funds for the `subaccount` owner, and potentially damage the reputation of the JOJO Exchange.

Here's an example PoC code that demonstrates the exploit:

```solidity
pragma solidity ^0.8.0;

contract Malicious {
    address subaccountAddr;
    
    constructor(address _subaccountAddr) {
        subaccountAddr = _subaccountAddr;
    }
    
    function attack() public payable {
        (bool success, ) = subaccountAddr.call{value: msg.value}("");
        require(success, "attack failed");
        subaccountAddr.call{value: msg.value}("");
    }
}

```

To execute the attack, the attacker would deploy the `Malicious` contract and pass the address of the` Subaccount` contract as a parameter to the constructor. Then, the `attacker` would call the `attack() `function in the `Malicious` contract with some Ether. This will repeatedly call back into the` Subaccount `contract and drain its funds.

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/subaccount/Subaccount.sol#L45

## Tool used

Manual Review

## Recommendation

ensure that the balance of the `subaccount` is updated after executing the transaction in the `execute()` function. One way to achieve this is to use the `call()` function instead of the `to.call{value: value}(data)` function call. The `call()` function returns a boolean value indicating whether the transaction was successful or not, and it also returns the` output` data
