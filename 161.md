rvierdiiev

high

# It's possible to reset primaryCredit and secondaryCredit for insurance account

## Summary
When because of negative credit after liquidations of another accounts, insurance address doesn't pass `isSafe` check, then malicious user can call JOJOExternal.handleBadDebt and reset both primaryCredit and secondaryCredit for insurance account.
## Vulnerability Detail
`insurance` account is handled by JOJO team. Team is responsible to top up this account in order to cover losses. When [bad debt is handled](https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/impl/Perpetual.sol#L165), then its negative credit value is added to the insurance account. Because of that it's possible that primaryCredit of insurance account is negative and `Liquidation._isSafe(state, insurance) == false`.

Anyone can call [`JOJOExternal.handleBadDebt` function](https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/impl/JOJOExternal.sol#L56-L58).
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/lib/Liquidation.sol#L399-L418
```solidity
    function handleBadDebt(Types.State storage state, address liquidatedTrader)
        external
    {
        if (
            state.openPositions[liquidatedTrader].length == 0 &&
            !Liquidation._isSafe(state, liquidatedTrader)
        ) {
            int256 primaryCredit = state.primaryCredit[liquidatedTrader];
            uint256 secondaryCredit = state.secondaryCredit[liquidatedTrader];
            state.primaryCredit[state.insurance] += primaryCredit;
            state.secondaryCredit[state.insurance] += secondaryCredit;
            state.primaryCredit[liquidatedTrader] = 0;
            state.secondaryCredit[liquidatedTrader] = 0;
            emit HandleBadDebt(
                liquidatedTrader,
                primaryCredit,
                secondaryCredit
            );
        }
    }
```
So it's possible for anyone to call `handleBadDebt` for `insurance` address, once its primaryCredit is negative and `Liquidation._isSafe(state, insurance) == false`. This will reset both primaryCredit and secondaryCredit variables to 0 and break insurance calculations.
## Impact
Insurance primaryCredit and secondaryCredit variables are reset.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Do not allow `handleBadDebt` call with insurance address.