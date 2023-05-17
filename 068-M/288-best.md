0xPkhatri

high

# Missing Update for RiskParams.isRegistered Boolean in Operation#setPerpRiskParams Function

## Summary

In the Operation#setPerpRiskParams function, if the perp is already registered and the setPerpRiskParams function is called, it fails to update the boolean isRegistered to true.

## Vulnerability Detail

The setPerpRiskParams function is intended to set or update RiskParams in the perpRiskParams mapping. However, the function does not correctly update the isRegistered in the state.perpRiskParams[perp] mapping when the perp RiskParams are already registered.
Example:

- Suppose the perp and RiskParams are already registered, and the owner wants to set them again, using [setPerpRiskParams](https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/impl/JOJOOperation.sol#L32-L37) which calls [Operation.setPerpRiskParams(state, perp, param)](https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/lib/Operation.sol#L45-L75)
- The setPerpRiskParams function starts by checking if the market is already registered

```solidity
if (state.perpRiskParams[perp].isRegistered && !param.isRegistered)

where,
state.perpRiskParams[perp].isRegistered = true
param.isRegistered = false // param.isRegistered is initially set to false, causing the if statement to execute

```
- The perp is already registered and needs to be removed. It does this by iterating through the state.registeredPerp array, replacing the target perp with the last element in the array, and then removing the last element.
- Finally, the function updates the risk parameters for the given perp in the state.perpRiskParams mapping.

```solidity
state.perpRiskParams[perp] = param;
```

- However, here param.isRegistered = false and the owner neglects to update param.isRegistered = true

## Impact

This results in an incorrect param.isRegistered boolean status and DOS the system.

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/impl/JOJOOperation.sol#L32-L37
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/lib/Operation.sol#L45-L75

## Tool used

Manual Review

## Recommendation

After setting RiskParams, update param.isRegistered to true:

```diff
function setPerpRiskParams(
    Types.State storage state,
    address perp,
    Types.RiskParams calldata param
) external {
    if (state.perpRiskParams[perp].isRegistered && !param.isRegistered) {
        // remove perp
        for (uint256 i; i < state.registeredPerp.length;) {
            if (state.registeredPerp[i] == perp) {
                state.registeredPerp[i] = state.registeredPerp[
                    state.registeredPerp.length - 1
                ];
                state.registeredPerp.pop();
            }
            unchecked {
                ++i;
            }
        }
    }
    if (!state.perpRiskParams[perp].isRegistered && param.isRegistered) {
        // new perp
        state.registeredPerp.push(perp);
    }
    require(
        param.liquidationPriceOff + param.insuranceFeeRate <=
            param.liquidationThreshold,
        Errors.INVALID_RISK_PARAM
    );
    state.perpRiskParams[perp] = param;
+   state.perpRiskParams[perp].isRegistered = true;
    emit UpdatePerpRiskParams(perp, param);
}
```
