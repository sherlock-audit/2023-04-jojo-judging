0xbepresent

high

# The JUSDBank depositor can't withdraw his deposited collateral excedent once the owner delisted a collateral token

## Summary

The depositor can't withdraw a portion of his collateral deposited amount once the collateral is delisted.

## Vulnerability Detail

An user can deposits his collateral via the [JUSDBank.deposit()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L93) function and then get a `JUSD` borrow via [JUSDBank.borrow()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L105) function.

The problem is that an user can get a small borrow amount of `JUSD` compared to the amount deposited as collateral, then the owner delists the collateral via [JUSDOperation.delistReserve()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDOperation.sol#LL227C14-L227C27) function and then the user can not withdraw a portion of his collateral even if the borrow is too small.

Please see the next scenario:
1. Alice deposits `100e18` mockToken1 as collateral.
2. Alice borrows only `1e6` JUSD.
3. Owner delists the mockToken1 as approved collateral via [JUSDOperation.delistReserve()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDOperation.sol#LL227C14-L227C27) function.
4. Alice withdraws `1e18` mockToken1 but is not possible because the transaction is reverted by `AFTER_WITHDRAW_ACCOUNT_IS_NOT_SAFE`.

The account is safe because Alice deposits `100e18` as collateral, she borrows only `1 JUSD` and she is denied by withdraw only `1e18` amount which make no sense because she has `100e18` as collateral.

I created a test with this behaviour:

```solidity
File: JUSDBankWithdraw.t.sol
44:     function testWithdrawDelistedCollateral() public {
45:         // After owner delists a collateral, the depositors are unable to withdraw their excedent amounts
46:         // 1. Alice deposits 100e18 mockToken1 as collateral.
47:         // 2. Alice borrows only 1e6 JUSD.
48:         // 3. Owner delists the mockToken1 as approved collateral.
49:         // 4. Alice withdraws 1e18 mockToken1 but is not possible because a revert "AFTER_WITHDRAW_ACCOUNT_IS_NOT_SAFE".
50:         //
51:         // 1. Alice deposits 100e18 mockToken1 as collateral.
52:         //
53:         mockToken1.transfer(alice, 100e18);
54:         vm.startPrank(alice);
55:         mockToken1.approve(address(jusdBank), 100e18);
56:         jusdBank.deposit(alice, address(mockToken1), 100e18, alice);
57:         //
58:         // 2. Alice borrows only 1e6 JUSD.
59:         //
60:         jusdBank.borrow(1e6, alice, false);
61:         vm.stopPrank();
62:         //
63:         // 3. Owner delists the mockToken1 as approved collateral.
64:         //
65:         jusdBank.delistReserve(address(mockToken1));
66:         //
67:         // 4. Alice withdraws 1e18 mockToken1 but is not possible because a revert "AFTER_WITHDRAW_ACCOUNT_IS_NOT_SAFE".
68:         //
69:         vm.startPrank(alice);
70:         cheats.expectRevert("AFTER_WITHDRAW_ACCOUNT_IS_NOT_SAFE");
71:         jusdBank.withdraw(address(mockToken1), 1e18, alice, false);
72:         vm.stopPrank();
73:     }
```

## Impact

The depositor is not allowed to withdraw any amount of his collateral even when he borrows only a small amount of JUSD. Depositor is unable to get his collateral.

The reason is because the [JUSDView._maxMintAmount()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDView.sol#L139) function does not contabilize the depositor collateral reserve that was delisted by the protocol owner.

```solidity
File: JUSDView.sol
139:     function _maxMintAmount(
140:         DataTypes.UserInfo storage user
141:     ) internal view returns (uint256) {
142:         address[] memory collaterals = user.collateralList;
143:         uint256 maxMintAmount;
144:         for (uint256 i; i < collaterals.length; i = i + 1) {
145:             address collateral = collaterals[i];
146:             DataTypes.ReserveInfo memory reserve = reserveInfo[collateral];
147:             if (!reserve.isBorrowAllowed) {
148:                 continue;
149:             }
150:             maxMintAmount += _getMintAmount(
151:                 user.depositBalance[collateral],
152:                 reserve.oracle,
153:                 reserve.initialMortgageRate
154:             );
155:         }
156:         return maxMintAmount;
157:     }
```

That is a contradiction because if the owner doesn't delist the collateral, the depositor can withdraw an excedent amount of his collateral as long as he has enough collateral.

The next test shows when the owner doesn't delists the collateral, the depositor can borrow a small amount of `JUSD` then he can withdraw a portion of is collateral.

```solidity
File: JUSDBankWithdraw.t.sol
77:     function testWithdrawCollateral() public {
78:         // 1. Alice deposits 100e18 mockToken1 as collateral.
79:         // 2. Alice borrows only 1e6 JUSD.
80:         // 3. Alice withdraws 10e18 mockToken1.
81:         //
82:         // 1. Alice deposits 100e18 mockToken1 as collateral.
83:         //
84:         mockToken1.transfer(alice, 100e18);
85:         vm.startPrank(alice);
86:         mockToken1.approve(address(jusdBank), 100e18);
87:         jusdBank.deposit(alice, address(mockToken1), 100e18, alice);
88:         //
89:         // 2. Alice borrows only 1e6 JUSD.
90:         //
91:         jusdBank.borrow(1e6, alice, false); 
92:         vm.stopPrank();
93:         //
94:         // 3. Alice withdraws 10e18 mockToken1.
95:         //
96:         vm.startPrank(alice);
97:         jusdBank.withdraw(address(mockToken1), 10e18, alice, false);
98:         vm.stopPrank();
99:     }
```

## Code Snippet

- The [JUSDBank.withdraw()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L128) function.
- The [JUSDView._isAccountSafe()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDView.sol#L123) that validates the amount available to withdraw in the [_maxMintAmount()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDView.sol#L139) function.

## Tool used

Manual review

## Recommendation

Even when the collateral was delisted, the depositor should be able to withdraw a portion of his collateral as long as he has enough collateral to cover the borrow.