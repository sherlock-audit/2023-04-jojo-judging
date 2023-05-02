0xDACA

high

# Multiple contracts with arbitrary calls leads to stolen funds



## Summary

The following contracts have arbitrary calls which allow an attacker to steal all tokens from them.

- `DepositStableCoinToDealer`, vulnerable function: `depositStableCoin`
- `FlashLoanLiquidate`, vulnerable function: `JOJOFlashLoan`
- `FlashLoanRepay`, vulnerable function: `JOJOFlashLoan`
- `GeneralRepay`, vulnerable function: `repayJUSD`

*Disclaimer:* If these contracts are meant to not hold any tokens and this is intentional, then you can probably discard this report.

## Vulnerability Detail

All the mentioned contracts have the following code:
```solidity
(bool success, ) = swapTarget.call(data);
```
Where both `swapTarget` and `data` are attacker controlled. An attacker can easily exploit this by

1. Setting `swapTarget` to be the ERC20 token he wants to steal
2. Setting `data` to be `approve(attacker, type(uint256).max)`

**Now the attacker can just use all the contract's tokens on his behalf!**

Note that since the attacker controls all parameters of the function, and there's no other reason for the contract to revert, so this attack works.

## Impact

Loss of all funds in the mentioned contracts.

## Code Snippet

DepositStableCoinToDealer:
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/stableCoin/DepositStableCoinToDealer.sol#L36

FlashLoanLiquidate:
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/flashloanImpl/FlashLoanLiquidate.sol#L62

FlashLoanRepay:
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/flashloanImpl/FlashLoanRepay.sol#L44

GeneralRepay:
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/flashloanImpl/GeneralRepay.sol#L44

## Tool used

Manual Review

## Recommendation

Instead of letting the user call whatever contract he wants with any arguments, call the specific contract that implements the intended logic, with set arguments.