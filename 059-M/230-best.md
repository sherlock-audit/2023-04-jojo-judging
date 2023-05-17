0xbepresent

medium

# When the `JUSDBank.withdraw()` is to another internal account the `ReserveInfo.isDepositAllowed` is not validated

## Summary

The [internal withdraw](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L348) does not validate if the collateral reserve has activated/deactivated the [isDepositAllowed](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/lib/DataTypes.sol#LL37C14-L37C30) variable

## Vulnerability Detail

The [JUSDBank.withdraw()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L128) function has a param called [isInternal](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L132) that helps to indicate if the withdraw amount is internal between accounts or not. When the withdraw is internal the `ReserveInfo.isDepositAllowed` is not validated.

```solidity
File: JUSDBank.sol
332:     function _withdraw(
333:         uint256 amount,
334:         address collateral,
335:         address to,
336:         address from,
337:         bool isInternal
338:     ) internal {
...
...
348:         if (isInternal) {
349:             DataTypes.UserInfo storage toAccount = userInfo[to];
350:             _addCollateralIfNotExists(toAccount, collateral);
351:             toAccount.depositBalance[collateral] += amount;
352:             require(
353:                 toAccount.depositBalance[collateral] <=
354:                     reserve.maxDepositAmountPerAccount,
355:                 JUSDErrors.EXCEED_THE_MAX_DEPOSIT_AMOUNT_PER_ACCOUNT
356:             );
...
...
```

In the other hand, the `isDepositAllowed` is validated in the [deposit](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L247) function in the `code line 255` but the withdraw to internal account is not validated.

```solidity
File: JUSDBank.sol
247:     function _deposit(
248:         DataTypes.ReserveInfo storage reserve,
249:         DataTypes.UserInfo storage user,
250:         uint256 amount,
251:         address collateral,
252:         address to,
253:         address from
254:     ) internal {
255:         require(reserve.isDepositAllowed, JUSDErrors.RESERVE_NOT_ALLOW_DEPOSIT);
```

Additionally, the `ReserveInfo.isDepositAllowed` can be modified via the [JUSDOperation.delistReserve()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDOperation.sol#L227) function. So any collateral's deposits can be deactivated at any time.

```solidity
File: JUSDOperation.sol
227:     function delistReserve(address collateral) external onlyOwner {
228:         DataTypes.ReserveInfo storage reserve = reserveInfo[collateral];
229:         reserve.isBorrowAllowed = false;
230:         reserve.isDepositAllowed = false;
231:         reserve.isFinalLiquidation = true;
232:         emit RemoveReserve(collateral);
233:     }
```

## Impact

The collateral's reserve can get deposits via the [internal withdraw](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L348-L356) even when the `Reserve.isDepositAllowed` is turned off making the `Reserve.isDepositAllowed` useless because the collateral deposits can be via `internal withdrawals`.

## Code Snippet

The [internal withdraw code](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L348-L356)

## Tool used

Manual review

## Recommendation

Add a `Reserve.isDepositAllowed` validation when the withdrawal is to another internal account.

```diff
File: JUSDBank.sol
    function _withdraw(
        uint256 amount,
        address collateral,
        address to,
        address from,
        bool isInternal
    ) internal {
...
...
        if (isInternal) {
++          require(reserve.isDepositAllowed, JUSDErrors.RESERVE_NOT_ALLOW_DEPOSIT);
            DataTypes.UserInfo storage toAccount = userInfo[to];
            _addCollateralIfNotExists(toAccount, collateral);
            toAccount.depositBalance[collateral] += amount;
            require(
                toAccount.depositBalance[collateral] <=
                    reserve.maxDepositAmountPerAccount,
                JUSDErrors.EXCEED_THE_MAX_DEPOSIT_AMOUNT_PER_ACCOUNT
            );
...
...
```