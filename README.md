# Issue H-1: All allowances to DepositStableCoinToDealer and GeneralRepay can be stolen due to unsafe call 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/428 

## Found by 
0x52, 0xkazim, ArbitraryExecution, Bauer, BowTiedOriole, GalloDaSballo, Inspex, Jigsaw, MalfurionWhitehat, XDZIBEC, ctf\_sec, dingo, jprod15, lil.eth, tvdung94, yy
## Summary

DepositStableCoinToDealer.sol and GeneralRepay.sol are helper contracts that allow a user to swap and enter JOJODealer and JUSDBank respectively. The issue is that the call is unsafe allowing the contract to call the token contracts directly and transfer tokens from anyone who has approved the contract.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/stableCoin/DepositStableCoinToDealer.sol#L30-L44

        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
        (address approveTarget, address swapTarget, bytes memory data) = abi
        .decode(param, (address, address, bytes));
        // if usdt
        IERC20(asset).approve(approveTarget, 0);
        IERC20(asset).approve(approveTarget, amount);
        (bool success, ) = swapTarget.call(data);
        if (success == false) {
            assembly {
                let ptr := mload(0x40)
                let size := returndatasize()
                returndatacopy(ptr, 0, size)
                revert(ptr, size)
            }
        }

We can see above that the call is totally unprotected allowing a user to make any call to any contract. This can be abused by calling the token contract and using the allowances of others. The attack would go as follows:

1. User A approves the contract for 100 USDT
2. User B sees this approval and calls depositStableCoin with the swap target as the USDT contract with themselves as the receiver
3. This transfers all of user A USDT to them

## Impact

All allowances can be stolen

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/stableCoin/DepositStableCoinToDealer.sol#L23C14-L50

## Tool used

Manual Review

## Recommendation

Only allow users to call certain whitelisted contracts.



## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/JUSDV1/commit/5770d15edac41c78d9726f02e988aa8e14601f3e
https://github.com/JOJOexchange/smart-contract-EVM/commit/94ea554ec1c563e945bd388051f6438826818b47

# Issue M-1: Changing the borrowFeeRate results in a lower than expectet borrowFeeRate for new borrowed JUSD 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/39 

## Found by 
BenRai
## Summary
If the JUSDBank changes the `borrowFeeRate`, the fee rate users are paying for borrowed JUSD is lower than the new `borrowFeeRate`

## Vulnerability Detail
Lets say the `borrowFeeRate` is 2% (2e16) and after a year the bank decides to change the borrowFeeRate to 1% (1e16). For this they call the function `jusdBank.updateBorrowFeeRate()`. This function sets the `lastUpdateTimestamp` to the current `block.timestamp` and sets the `t0Rate` to the current `TRate`. 
After one year the `TRate` is `1020000000000000000`. This means, when updating the `borrowFeeRate`, ` t0Rate` is set to `1020000000000000000`. 

## Impact

After changing the  `borrowFeeRate` after one year from 2% to 1%,  a user borrows 100 JUSD. When repaying this 100 JUSD after 1 year, the bank would expect to get 101 JUSD but since the base of the borrowing fee calculation (` t0Rate`) is no longer 1 but 1,02 the repayment amount for 100 JUSD after one year is only 100.980392 JUSD instead of 101 JUSD. This leads to a nearly 2% lost of borrowing fee income for the bank. This effect gets even worse with each change of the borrowFeeRate since the base of the borrowing fee calculation (` t0Rate`) becomes bigger each time. Over time this effect will lead to millions of JUSD in borrowing fee less paid to the bank. 


## POC
### Note: for the POC to work I increased the `heartbeatInterval` by a lot to avoid the error `ORACLE_HEARTBEAT_FAILED`


// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.9;

import "./../JUSDBankInit.t.sol";
import "../../mocks/MockJOJODealerRevert.sol";
import "../../mocks/MockChainLink15000.sol";

contract JUSDBankBorrowTest is JUSDBankInitTest {
    MockJOJODealerRevert public jojoDealerRevert = new MockJOJODealerRevert();

        function testBorrowWithLessIntrest() public {
        mockToken1.transfer(alice, 100e18);
        console.log("BorrowFeeRate in %", jusdBank.borrowFeeRate()); // 2%
        skip(365* 86_400); // skip 1 year
        console.log("TRate after 1 year", jusdBank.getTRate());
        console.log("T0Rate after 1 year", jusdBank.t0Rate());
        jusdBank.updateBorrowFeeRate(10000000000000000); // change BorrowFeeRate to 1% => t0Rate = 1,02
        console.log("T0Rate after update BorrowFeeRate", jusdBank.t0Rate());
        
        vm.startPrank(alice);
        mockToken1.approve(address(jusdBank), 10e18);
        jusdBank.deposit(alice, address(mockToken1), 10e18, alice);
        jusdBank.borrow(100e6, alice, false); // borrow 1JUSD for 1% interest
        console.log("Borrowed balance before time skip in JUSD", jusdBank.getBorrowBalance(alice)); // 100 JUSD
        skip(365* 86_400); // skip 1 year
        // would expect 1,01 JUSD but is 1,0098 JUSD
        console.log("Borrowed balance after time skip in JUSD", jusdBank.getBorrowBalance(alice)); 
        vm.stopPrank();

        assertEq(jusdBank.getBorrowBalance(alice), 100980392);
    }


}





## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/6090fef68932b5577abf6b5aa26eb1e579353c57/JUSDV1/src/Impl/JUSDOperation.sol#L168-L173


## Tool used
VSC

## Recommendation

To make sure the fees received by the bank match the `borrowFeeRate`, the base of the fee calculation (` t0Rate`)  for JUSD borrowed after changing should be adjusted so it is 1 at the time of borrowing. This could be done by creating an array of loans for each user where each loan includes the block.timestamp when the laon was initiated, the loan amount and an adjustment factor that modifies the ` t0Rate` so it is 1 at the time the loan was given. Eg. for the example above:

struct Loan{
uint256 amount; //eg. 100
uint256 timeBorrowed; // eg.  time loan was given
uint256 t0RateAndjustment // eg. `tRate when loan was given - 1`
}

For calculating the BorrowBalance one would need to loop over the array of loans and

- calculate the borrowBalance for each loan using `timeBorrowed` instead of `lastUpdateTimestamp` and adjusting the current tRate buy `t0RateAndjustment `






## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/JUSDV1/commit/334fb691eea57a96fb7220e67e31517638725a80

# Issue M-2: If user has to many open positions, the function `getTotalExposure()` will fail because it runs out of gas 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/47 

## Found by 
BenRai, MalfurionWhitehat, \_\_141345\_\_, moneyversed, yixxas
## Summary

If a user has to many open positions the function `getTotalExposure()` will run out of gas while calculating the `netPositionValue` and looping over ` openPositions[trader]`. This will result in the failure of all functions that call the function `getTotalExposure()`.

## Vulnerability Detail

The functions ` _isSafe ` and `_isSolidSafe` are calling `getTotalExposure()` and can therefore run out of gas. 

 The following “end functions” are affected by this and will also run out of gas:

`Perpetual.trade()`,
`Perpetual. liquidate()` (trough `Liquidation.handleBadDebt()`),
‘Funding. executeWithdraw()’ (through `Funding. _withdraw`),
`Liquidation.requestLiquidation()` (trough `Liquidation.getLiquidateCreditAmount()`)
`JOJOExternal. approveTrade()`



## Impact

A user that has to many open positions can not withdraw any assets even if he would meet the necessary criteria.  (‘Funding. executeWithdraw()’ is failing)  He would be forced to close positions to free his assets again by using the function `_realizePnl`. Since this function `_realizePnl` also runs a loop over all open Positions of the user  to find the position that should be closed, the user would need to close one of his oldest positions to avoid running out of gas here also. If the user has eg. 100 positions open and he would need e.g. 90 to be able withdraw his assets he would be forced to close 10 positions. If the positions that he can close without running out of gas using the function `_realizePnl`  are all out of profit, he might be forced to realize loses. This might lead to him leaving the JOJO ecosystem. This is especially bad for the exchange since individuals who have a lot of open positions are the once that trade a lot and bring in the most fee revenues.


## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/6090fef68932b5577abf6b5aa26eb1e579353c57/smart-contract-EVM/contracts/lib/Liquidation.sol#L63-L84


## Tool used

Manual Review

## Recommendation

Set a cap of open positions one address can have. If the user wands to open more positions he can create a sub account and open new positions there. 



## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/smart-contract-EVM/commit/fbee64482004062de5168c7e0b81525b2a1c1483

# Issue M-3: Multiple contracts with arbitrary calls leads to stolen funds 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/72 

## Found by 
0xDACA, BugHunter101, ashirleyshe, dingo


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



## Discussion

**JoscelynFarr**

These contracts is considered not to hold any tokens

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/JUSDV1/commit/1e281f99b027b6df7814575d1ac1e0df63104e36

# Issue M-4: JUSD borrow fee rate is less than it should be 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/73 

## Found by 
Ruhum, carrotsmuggler, y1cunhui
## Summary
The borrow fee rate calculation is wrong causing the protocol to take less fees than it should.

## Vulnerability Detail
The borrowFeeRate is calculated through `getTRate()`:

```sol
    function getTRate() public view returns (uint256) {
        uint256 timeDifference = block.timestamp - uint256(lastUpdateTimestamp);
        return
            t0Rate +
            (borrowFeeRate * timeDifference) /
            JOJOConstant.SECONDS_PER_YEAR;
    }
```
`t0Rate` is initialized as `1e18` in the test contracts:

```sol
    constructor(
        uint256 _maxReservesNum,
        address _insurance,
        address _JUSD,
        address _JOJODealer,
        uint256 _maxPerAccountBorrowAmount,
        uint256 _maxTotalBorrowAmount,
        uint256 _borrowFeeRate,
        address _primaryAsset
    ) {
        // ...
        t0Rate = JOJOConstant.ONE;
    }
```

`SECONDS_PER_YEAR` is equal to `365 days` which is `60 * 60 * 24 * 365 = 31536000`:

```sol
library JOJOConstant {
    uint256 public constant SECONDS_PER_YEAR = 365 days;
}
```

As time passes, `getTRate()` value will increase. When a user borrows JUSD the contract doesn't save the actual amount of JUSD they borrow, `tAmount`. Instead, it saves the current "value" of it, `t0Amount`:

```sol
    function _borrow(
        DataTypes.UserInfo storage user,
        bool isDepositToJOJO,
        address to,
        uint256 tAmount,
        address from
    ) internal {
        uint256 tRate = getTRate();
        //        tAmount % tRate ？ tAmount / tRate + 1 ： tAmount % tRate
        uint256 t0Amount = tAmount.decimalRemainder(tRate)
            ? tAmount.decimalDiv(tRate)
            : tAmount.decimalDiv(tRate) + 1;
        user.t0BorrowBalance += t0Amount;
```

When you repay the JUSD, the same calculation is done again to decrease the borrowed amount. Meaning, as time passes, you have to repay more JUSD.

Let's say that JUSDBank was live for a year with a borrowing fee rate of 10% (1e17). `getTRate()` would then return:
$1e18 + 1e17 * 31536000 / 31536000 = 1.1e18$

If the user now borrows 1 JUSD we get: $1e6 * 1e18 / 1.1e18 ~= 909091$ for `t0Amount`. That's not the expected 10% decrease. Instead, it's about 9.1%.

## Impact
Users are able to borrow JUSD for cheaper than expected

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L274-L286
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/lib/JOJOConstant.sol#L7
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L37
## Tool used

Manual Review

## Recommendation
Change formula to:
`t0Amount = tAmount - tAmount.decimalMul(tRate)` where `t0Rate` is initialized with `0` instead of `1e18`.



## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/JUSDV1/commit/334fb691eea57a96fb7220e67e31517638725a80

# Issue M-5: Missing checks for whether Arbitrum Sequencer is active 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/101 

## Found by 
0x52, 0xGoodess, 0xPkhatri, 0xStalin, 0xhacksmithh, 0xkazim, 0xrobsol, AlexCzm, Avci, Bauer, Brenzee, ChainGuardian, GalloDaSballo, GimelSec, HexHackers, J4de, Saeedalipoor01988, T1MOH, ast3ros, aviggiano, bytes032, deadrxsezzz, immeas, jasonxiale, kaysoft, p0wd3r, peakbolt, tallo, yixxas


## Summary

Missing checks for whether Arbitrum Sequencer is active

## Vulnerability Detail

the onchain deployment context is changed, in prev contest the protocol only attemps to deploy the code to ethereum while in the current contest

the protocol intends to deploy to arbtrium as well!

Chainlink recommends that users using price oracles, check whether the Arbitrum sequencer is active

[https://docs.chain.link/data-feeds#l2-sequencer-uptime-feeds](https://docs.chain.link/data-feeds#l2-sequencer-uptime-feeds)

If the sequencer goes down, the index oracles may have stale prices, since L2-submitted transactions (i.e. by the aggregating oracles) will not be processed.

## Impact

Stale prices, e.g. if USDC were to de-peg while the sequencer is offline, stale price is used and can result in false liquidation or over-borrowing.

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/oracle/JOJOOracleAdaptor.sol#L26-L35

```solidity
    function getAssetPrice() external view override returns (uint256) {
        /*uint80 roundID*/
        (, int256 price,, uint256 updatedAt,) = IChainLinkAggregator(chainlink).latestRoundData();
        (, int256 USDCPrice,, uint256 USDCUpdatedAt,) = IChainLinkAggregator(USDCSource).latestRoundData();

        require(block.timestamp - updatedAt <= heartbeatInterval, "ORACLE_HEARTBEAT_FAILED");
        require(block.timestamp - USDCUpdatedAt <= heartbeatInterval, "USDC_ORACLE_HEARTBEAT_FAILED");
        uint256 tokenPrice = (SafeCast.toUint256(price) * 1e8) / SafeCast.toUint256(USDCPrice);
        return tokenPrice * JOJOConstant.ONE / decimalsCorrection;
    }
```

## Tool used

Manual Review

## Recommendation

Use sequencer oracle to determine whether the sequencer is offline or not, and don't allow orders to be executed while the sequencer is offline.



## Discussion

**Trumpero**

During the time when the sequencer is down, transactions should be reverted to prevent users from experiencing unexpected prices once the sequencer is up again. Therefore, I believe this issue should be classified as medium severity.

# Issue M-6: Subaccount#execute lacks payable 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/111 

## Found by 
0xlmanini, Brenzee, T1MOH, ctf\_sec, immeas, n1punp, nobody2018, p0wd3r, rvierdiiev, yixxas
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



## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/smart-contract-EVM/commit/64dfd055deeae857fa99d4703cdbf7ba1291b8ad

# Issue M-7: Attacker can frontrun SubaccountFactory.newSubaccount and steal user funds 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/141 

## Found by 
rvierdiiev
## Summary
Because `SubaccountFactory.newSubaccount` creates account using `create` function, that means that attacker can frontrun the call in order to get control under that account. And if victim has sent any token or made approvals, then attacker can steal them.
## Vulnerability Detail
`SubaccountFactory.newSubaccount` creates new subaccount for the `msg.sender`. It uses [`create` function to do that](https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/subaccount/SubaccountFactory.sol#L40).
Once user has created subaccount then he can create some token approvals for that subaccount to be able to open positions.
It's possible that user will make this 2 tx one by one.
1.He creates subaccount.
2.He approves USDC to the subaccount.

Attacker can watch to such transactions and when he detects them, then he reorders in following way.
1.Attacker creates subaccount(so now he controls that account address as `create` is used).
2.Victim approves USDC to attackers subaccount.

As result, victim has approved subaccount where owner is attacker. So now attacker can use `Subaccount.execute` in order to steal those tokens.
## Impact
Attacker can steal funds.
## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/subaccount/SubaccountFactory.sol#L40-L41
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/subaccount/Subaccount.sol#L45-L58
## Tool used

Manual Review

## Recommendation
Create subaccount using `create2` with salt that contains `msg.sender` info.



## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/smart-contract-EVM/commit/7e2a8fe43ad3280246f71fbc7b4aacd8020276dd

# Issue M-8: It's possible to reset primaryCredit and secondaryCredit for insurance account 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/159 

## Found by 
GalloDaSballo, p0wd3r, rvierdiiev
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



## Discussion

**JoscelynFarr**

fix link: 
https://github.com/JOJOexchange/smart-contract-EVM/commit/78c53b4721ae7bb97fb922f78342d0ee4a1825dd

# Issue M-9: Missing check of oracle's return price 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/173 

## Found by 
0x2e, 0xChinedu, 0xhacksmithh, Aymen0909, Bauchibred, Bauer, Brenzee, Delvir0, HexHackers, J4de, MalfurionWhitehat, MohammedRizwan, Ruhum, Saeedalipoor01988, T1MOH, ast3ros, branch\_indigo, bytes032, caventa, deadrxsezzz, georgits, holyhansss, kaysoft, koxuan, m9800, martin, peanuts, tsvetanovv, volodya, w42d3n
## Summary
Missing check the parameter rawPrice return from chainlink oracle


## Vulnerability Detail
The lastRoundData()'s parameters according to [Chainlink](https://docs.chain.link/docs/data-feeds/price-feeds/api-reference/) are the following:
```solidity
function latestRoundData() external view
    returns (
        uint80 roundId,             //  The round ID.
        int256 answer,              //  The price.
        uint256 startedAt,          //  Timestamp of when the round started.
        uint256 updatedAt,          //  Timestamp of when the round was updated.
        uint80 answeredInRound      //  The round ID of the round in which the answer was computed.
    )
```
Function ChainlinkExpandAdaptor.getMarkPrice() has 1 checks with 2 parameters price, updatedAt but forget to check the price return value which lead to the 0 price data from chainlink.
```solidity
function getMarkPrice() external view returns (uint256 price) {
        int256 rawPrice;
        uint256 updatedAt;
        (, rawPrice, , updatedAt, ) = IChainlink(chainlink).latestRoundData();
        (, int256 USDCPrice,, uint256 USDCUpdatedAt,) = IChainlink(USDCSource).latestRoundData();
        require(
            block.timestamp - updatedAt <= heartbeatInterval,
            "ORACLE_HEARTBEAT_FAILED"
        );
        require(block.timestamp - USDCUpdatedAt <= heartbeatInterval, "USDC_ORACLE_HEARTBEAT_FAILED");
        uint256 tokenPrice = (SafeCast.toUint256(rawPrice) * 1e8) / SafeCast.toUint256(USDCPrice);
        return tokenPrice * 1e18 / decimalsCorrection;
    }
```

## Impact
Incorrect return price value lead to incorrect de-peg events trigger.


## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/adaptor/chainlinkAdaptor.sol#L43-L55

## Tool used

Manual Review

## Recommendation
Modify the function to have a check like this:
```solidity
(uint80 roundID, int256 rawPrice, , uint256 updatedAt, uint80 answeredInRound) = feed.latestRoundData();
require(rawPrice> 0, "Chainlink price <= 0"); 
require(answeredInRound >= roundID, "Stale price");
 require(
            block.timestamp - updatedAt <= heartbeatInterval,
            "ORACLE_HEARTBEAT_FAILED"
        );
```

# Issue M-10: Unable to liquidate USDC blacklisted user's loan due to transferring leftover collateral back in USDC 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/206 

## Found by 
Inspex, jprod15, m9800, monrel, peakbolt

## Summary
During the loan liquidation process, any remaining collateral will be swapped to USDC tokens and transferred to the liquidated user. However, if the USDC contract blacklists the liquidated user, the liquidation transaction will be revert. As a result, the user's loan will be unable to be liquidated if they have been blacklisted by the USDC token contract.


## Vulnerability Detail

During the liquidation process, any remaining tokens will be transferred to the owner of the loan. However, if the loan owner has been blacklisted by USDC token, this flow will be reverted due to the code shown below.

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L199-L204

As a result, users who have been blacklisted by USDC will be unable to liquidate their loan positions during the period of the blacklisting.

## Impact
The liquidation process might DoS due to its reliance on paying back remaining tokens in USDC only. This will error where transferring USDC tokens to blacklisted users can cause the transaction to be reverted, disrupting the liquidation flow. This will result in a bad debt for the platform.

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L199-L204

## Tool used

Manual Review

## Recommendation
We suggest implementing one or all of the following solutions:
1. Prevent USDC blacklisted users from opening a loan position until they are no longer blacklisted. This can be done by implementing a blacklist check during the borrowing process.
2. Remove the transfer of remaining USDC tokens to the liquidated user during the liquidation flow. Instead, allow the user to withdraw their remaining USDC tokens on their own after the liquidation process is complete.



## Discussion

**JoscelynFarr**

We will allow partial liquidation to avoid this happened.

# Issue M-11: `JUSDBank.sol#_calculateLiquidateAmount` user can front-run liquidate 1 token to prevent others from liquidating 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/221 

## Found by 
0x007, J4de, carrotsmuggler
## Summary

`JUSDBank.sol#_calculateLiquidateAmount` user can front-run repay by 1 token to prevent others from liquidating

## Vulnerability Detail

```solidity
File: Impl/JUSDBank.sol
379     function _calculateLiquidateAmount(
380         address liquidated,
381         address collateral,
382         uint256 amount
383     ) internal view returns (DataTypes.LiquidateData memory liquidateData) {
384         DataTypes.UserInfo storage liquidatedInfo = userInfo[liquidated];
385         require(amount != 0, JUSDErrors.LIQUIDATE_AMOUNT_IS_ZERO);
386         require(
387 >>          amount <= liquidatedInfo.depositBalance[collateral],
388             JUSDErrors.LIQUIDATE_AMOUNT_IS_TOO_BIG
389         );
```

If the liquidation amount is greater than the user's collateral amount during liquidation, revert will occur. Users can expoit this to prevent liquidation:

1. Bob has 100 * 1e18 ETH for collateral, and the position is liquidable
2. Alias is going to liquidate all of Bob's ETH, a total of 100 * 1e18
3. Bob discovered Alias' request, and the front-run liquidated 1 ETH
4. Liquidation fails because 100 * 1e18 > (100 * 1e18 - 1)

P.S. This prevents liquidation itself is invalid, because anyone can have unlimited addresses

```solidity
        require(
            liquidator != liquidated,
            JUSDErrors.SELF_LIQUIDATION_NOT_ALLOWED
        );
```

## Impact

Users can expoit it to prevent being liquidated

## Code Snippet

https://github.com/JOJOexchange/JUSDV1/blob/011e10d36257a404c8c1d7d2b8c9f01a2b7a1969/src/Impl/JUSDBank.sol#L387

## Tool used

Manual Review

## Recommendation

It is recommended to deal with the maximum value when it exceeds

```diff
    function _calculateLiquidateAmount(
        address liquidated,
        address collateral,
        uint256 amount
    ) internal view returns (DataTypes.LiquidateData memory liquidateData) {
        DataTypes.UserInfo storage liquidatedInfo = userInfo[liquidated];
        require(amount != 0, JUSDErrors.LIQUIDATE_AMOUNT_IS_ZERO);
-       require(
-           amount <= liquidatedInfo.depositBalance[collateral],
-           JUSDErrors.LIQUIDATE_AMOUNT_IS_TOO_BIG
-       );
+       if (amount > liquidatedInfo.depositBalance[collateral]) {
+           amount = liquidatedInfo.depositBalance[collateral];
+       }
```



## Discussion

**JoscelynFarr**

This is allowed. The purpose of liquidation is to return JUSD to make the account safe. If the user is partially liquidated, other liquidation cannot be performed.

**JoscelynFarr**

After internal discussion, we decide to ban this for DOS attack. Will keep it on medium level.

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/JUSDV1/commit/1a2c7654fabab7d3ec45a7f3e590202ba6779e9a

# Issue M-12: When the `JUSDBank.withdraw()` is to another internal account the `ReserveInfo.isDepositAllowed` is not validated 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/230 

## Found by 
0xbepresent, carrotsmuggler, caventa
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



## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/JUSDV1/commit/4071d470c126bac25b1a391d5dc1582db258280d

# Issue M-13: Later borrowers will pay less interest compared to earlier borrowers over the same period 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/246 

## Found by 
RaymondFam
## Summary
The way the interest calculation is designed, later borrowers will pay less interest compared to earlier borrowers over the same period, assuming `borrowFeeRate` remains unchanged.

## Vulnerability Detail
Suppose the first user borrowed JUSD at t0, the total debt plus interest say 10% per year incurred in 6 months is amount borrowed * 1.05/1, i.e. interest is 5%. If someone were to borrow at this time, the interest incurred 6 months later would be (1.1/1.05 - 1)*100% = 4.76%. This will be more and more significant down the road. For instance, borrowing 10 years later will have the interest (2/1.95 - 1)*100% = 2.56%. All future borrowers are paying less and lesser interest. 

## Impact
This could be seen as unfair to earlier borrowers and might not be the desired behavior for your system.

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBankStorage.sol#L51-L57

## Tool used

Manual Review

## Recommendation
One way to address this issue is to modify the interest calculation method to calculate interest rates based on the time each borrower took the loan. This can be done by storing the timestamp for each borrower when they take a loan and using it in the interest calculation.

For instance, you can create a new mapping to store the borrowing timestamp for each borrower:

```solidity
mapping(address => uint256) public borrowingTimestamps;
```
When a user borrows, update the timestamp:
```solidity
borrowingTimestamps[borrower] = block.timestamp;
```
And when calculating the interest rate for repayment, use the borrower's borrowing timestamp instead of the global `lastUpdateTimestamp`:
```solidity
uint256 timeDifference = block.timestamp - borrowingTimestamps[borrower];
```
This way, each borrower's interest rate will be calculated independently based on the time elapsed since they borrowed, ensuring fairness across all users.

Keep in mind that this approach will increase the storage requirements for your contract, as you'll need to store individual timestamps for each borrower. Additionally, it will require refactoring in other related codes. However, it ensures that each borrower pays interest based on their borrowing time, which can be considered a fairer approach.



## Discussion

**JoscelynFarr**

Hi we will update the borrow fee rate mechanism. We will change it to compound interest every `borrow`, `repay`, `liquidate` and `updateBorrowFeeRate`

**JoscelynFarr**

fix link: 
https://github.com/JOJOexchange/JUSDV1/commit/334fb691eea57a96fb7220e67e31517638725a80

# Issue M-14: Uniswap getting the price from all available pools for certain token pair possesses a risk 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/281 

## Found by 
ArbitraryExecution, GalloDaSballo, deadrxsezzz
## Summary
UniswapV3 oracle price can be easily manipulated

## Vulnerability Detail
Currently, the implementation of the UniswapV3 oracle gets the price of a token pair based on all available pools with said token pair.
```solidity
(uint256 uniswapPriceFeed,) = IStaticOracle(UNISWAP_V3_ORACLE).quoteAllAvailablePoolsWithTimePeriod(uint128(10**decimal), baseToken, quoteToken, period);
```
Uniswap allows for numerous pools to exist for the same token pair as long as they have different fee levels. For certain pairs (e.g. pools with 1% fee for stablecoins) some pools may exist, but have very little liquidity. This can make price manipulation very cheap , although TWAP is used.
An attacker could also deploy a pool for a certain fee level (to manipulate its price) when it does not exist already. 


## Impact
Uniswap oracle will return false data. A malicious user might manipulate Uniswap's oracle to return token's value to be higher than what it really is, allowing himself to profit off the protocol

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/adaptor/uniswapPriceAdaptor.sol#L49


## Tool used

Manual Review

## Recommendation
Either use specific pools to calculate the price of a token pair or add a requirement for liquidity pools to not have more than 50% difference in liquidity between eachother 




## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/smart-contract-EVM/commit/1dbc9001be667af42952c110e9fdf04fd7826669
https://github.com/JOJOexchange/JUSDV1/commit/eed86242c2be0cd70e6b412124eb05ed5e3c92dc

# Issue M-15: Lack of burn mechanism for JUSD repayments causes oversupply of JUSD 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/306 

## Found by 
peakbolt
## Summary

`JUSDBank.repay()` allow users to repay their JUSD debt and interest by transfering in JUSD tokens. Without a burn mechanism, it will cause an oversupply of JUSD that is no longer backed by any collateral.




## Vulnerability Detail

`JUSDBank` receives JUSD tokens for the repayment of debt and interest. However, there are no means to burn these tokens, causing JUSD balance in JUSDBank to keep increasing. 

That will lead to an oversupply of JUSD that is not backed by any collateral. And the oversupply of JUSD will increase significantly during market due to mass repayments from liquidation.


https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L307-L330
```Solidity
    function _repay(
        DataTypes.UserInfo storage user,
        address payer,
        address to,
        uint256 amount,
        uint256 tRate
    ) internal returns (uint256) {
        require(amount != 0, JUSDErrors.REPAY_AMOUNT_IS_ZERO);
        uint256 JUSDBorrowed = user.t0BorrowBalance.decimalMul(tRate);
        uint256 tBorrowAmount;
        uint256 t0Amount;
        if (JUSDBorrowed <= amount) {
            tBorrowAmount = JUSDBorrowed;
            t0Amount = user.t0BorrowBalance;
        } else {
            tBorrowAmount = amount;
            t0Amount = amount.decimalDiv(tRate);
        }
        IERC20(JUSD).safeTransferFrom(payer, address(this), tBorrowAmount);
        user.t0BorrowBalance -= t0Amount;
        t0TotalBorrowAmount -= t0Amount;
        emit Repay(payer, to, tBorrowAmount);
        return tBorrowAmount;
    }
```

## Impact

To maintain its stability, JUSD must always be backed by more than 1 USD worth of collateral. 

When there is oversupply of JUSD that is not backed by any collateral, it affects JUSD stability and possibly lead to a depeg event.



## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L307-L330


## Tool used
Manual review

## Recommendation
Instead of transfering to the JUSDBank upon repayment, consider adding a burn mechanism to reduce the supply of JUSD so that it will be adjusted automatically.



## Discussion

**JoscelynFarr**

Will add burn mechanism in the contract

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/JUSDV1/commit/a72604efba3a9cbce997aefde742be4c5036a039

# Issue M-16: A liquidated user can assign an operator, then the operator can liquidates him on behalf his client (the user who assigned him as operator), bypassing the `SELF_LIQUIDATION_NOT_ALLOWED` restriction 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/321 

## Found by 
0xbepresent, \_\_141345\_\_, branch\_indigo, immeas
## Summary

The liquidated user can assign an operator, then the operator can liquidate on behalf of the user who assigned him as operator. The liquidated user can get his own liquidated collateral.

## Vulnerability Detail

There is a validation by the [isValidLiquidator()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L365) function where it checks the liquidated address must not be the liquidator address in the 366 code line. So the liquidated address can not be the liquidator.

```solidity
File: JUSDBank.sol
365:     function isValidLiquidator(address liquidated, address liquidator) internal view {
366:         require(
367:             liquidator != liquidated,
368:             JUSDErrors.SELF_LIQUIDATION_NOT_ALLOWED
369:         );
370:         if(isLiquidatorWhitelistOpen){
371:             require(isLiquidatorWhiteList[liquidator], JUSDErrors.LIQUIDATOR_NOT_IN_THE_WHITELIST);
372:         }
373:     }
```

The problem is that the liquidated address can assign an operator, then the operator can liquidate on behalf the liquidated address.

I created the next test:

1. Alice deposit 10e18 mockToken1 and borrow 742e6 JUSD
2. Alice can be liquidated because the collateral price is updated
3. Alice tries to liquidate herself but the transaction is reverted by "SELF_LIQUIDATION_NOT_ALLOWED"
4. Alice assigns the operator "Jim".
5. Now Jim can liquidate Alice causing to Alice to get the collateral USDC. This bypass the restriction "SELF_LIQUIDATION_NOT_ALLOWED"

```solidity
File: JUSDBankLiquidateCollateral.t.sol
200:     function testSelfLiquidationViaOperator() public {
201:         // Self liquidation via the client's assigned operator
202:         // 1. Alice deposit 10e18 mockToken1 and borrow 742e6 JUSD
203:         // 2. Alice can be liquidated because the collateral price is updated
204:         // 3. Alice tries to liquidate herself but the transaction is reverted by "SELF_LIQUIDATION_NOT_ALLOWED"
205:         // 4. Alice assigns the operator "Jim".
206:         // 5. Now Jim can liquidate Alice causing to Alice to get the collateral USDC. This bypass the restriction "SELF_LIQUIDATION_NOT_ALLOWED"
207:         //
208:         // 1. Alice deposit 10e18 mockToken1 and borrow 742e6 JUSD
209:         //
210:         mockToken1.transfer(alice, 10e18);
211:         vm.startPrank(alice);
212:         mockToken1.approve(address(jusdBank), 10e18);
213:         jusdBank.deposit(alice, address(mockToken1), 10e18, alice);
214:         jusdBank.borrow(7426e6, alice, false);
215:         vm.stopPrank();
216:         //
217:         // 2. Alice can be liquidated because the collateral price is updated
218:         //
219:         MockChainLink900 eth900 = new MockChainLink900();
220:         JOJOOracleAdaptor jojoOracle900 = new JOJOOracleAdaptor(
221:             address(eth900),//source
222:             20,//decimal correction
223:             86400,//heartBeat
224:             address(usdcPrice)//USDSource
225:         );
226:         jusdBank.updateOracle(address(mockToken1), address(jojoOracle900));
227:         swapContract.addTokenPrice(address(mockToken1), address(jojoOracle900));
228:         FlashLoanLiquidate flashLoanLiquidate = new FlashLoanLiquidate(
229:             address(jusdBank),
230:             address(jusdExchange),
231:             address(USDC),
232:             address(jusd),
233:             insurance
234:         );
235:         bytes memory data = swapContract.getSwapData(
236:             10e18,
237:             address(mockToken1)
238:         );
239:         bytes memory param = abi.encode(
240:             swapContract,//approveTarget
241:             swapContract,//swapTarget
242:             address(alice),//liquidator
243:             data//data
244:         );
245:         vm.startPrank(alice);
246:         bytes memory afterParam = abi.encode(
247:             address(flashLoanLiquidate),
248:             param
249:         );
250:         //
251:         // 3. Alice tries to liquidate herself but the transaction is reverted by "SELF_LIQUIDATION_NOT_ALLOWED"
252:         //
253:         cheats.expectRevert("SELF_LIQUIDATION_NOT_ALLOWED");
254:         jusdBank.liquidate(
255:             alice, // Liquidated
256:             address(mockToken1),  //collateral used
257:             alice, //liquidator
258:             10e18, //10e18 amount
259:             afterParam,
260:             10e18 //expectedPrice
261:         );
262:         //
263:         // 4. Alice assigns the operator "Jim".
264:         //
265:         jusdBank.setOperator(jim, true);
266:         vm.stopPrank();
267:         console.log("Alice balance JUSD before liquidation:", IERC20(jusd).balanceOf(address(alice)));
268:         console.log("Alice balance USDC before liquidation:", IERC20(USDC).balanceOf(address(alice)));
269:         //
270:         // 5. Now Jim can liquidate Alice causing to Alice to get the collateral USDC. This bypass the restriction "SELF_LIQUIDATION_NOT_ALLOWED"
271:         //
272:         // Now Jim can liquidate his client Alice
273:         vm.startPrank(jim);
274:         jusdBank.liquidate(
275:             alice, // Liquidated
276:             address(mockToken1),  // collateral used
277:             jim, // liquidator Jim
278:             10e18, //10e18 amount
279:             afterParam,
280:             10e18 //expectedPrice
281:         );
282:         console.log("");
283:         console.log("Alice balance JUSD after liquidation: ", IERC20(jusd).balanceOf(address(alice)));
284:         console.log("Alice balance USDC after liquidation: ", IERC20(USDC).balanceOf(address(alice)));
285:     }
```

Output:

```bash
Running 1 test for test/Impl/JUSDBankLiquidateCollateral.t.sol:JUSDBankLiquidateCollateralTest
[PASS] testSelfLiquidationViaOperator() (gas: 1782422)
Logs:
  Alice balance JUSD before liquidation: 7426000000
  Alice balance USDC before liquidation: 0
  
  Alice balance JUSD after liquidation:  7426000000
  Alice balance USDC after liquidation:  748888889
```

## Impact

The liquidated user can get his own liquidated collateral, so the [SELF_LIQUIDATION_NOT_ALLOWED](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L366-L369) restriction is bypassed.

## Code Snippet

- The [JUSDBank.liquidate()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L143) function
- The [JUSDBank.isValidLiquidator()](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L365) function.

## Tool used

Manual review

## Recommendation

If the liquidated address assigns an operator, don't let the operator to liquidates the user who assigned him as operator.



## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/JUSDV1/commit/db907e6fdfa1c97d6e793c93cb4ff54e4ab7655c

# Issue M-17: Smart contract EVM: Malicious user can prevent liquidation by openning lots of positions due to gas limit 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/348 

## Found by 
Ace-30, Delvir0, J4de
## Summary
to liquidate an account one should call requestLiquidation()

- requestLiquidation() calls getLiquidateCreditAmount()
- getLiquidateCreditAmount() calls _isSafe()
- _isSafe() calls getTotalExposure()
- getTotalExposure() goes through all openPositions of trader
```solidity
    function getTotalExposure(Types.State storage state, address trader)
        public
        view
        returns (
            int256 netPositionValue,
            uint256 exposure,
            uint256 maintenanceMargin
        )
    {
        // sum net value and exposure among all markets
        for (uint256 i = 0; i < state.openPositions[trader].length; ) { 
            (int256 paperAmount, int256 creditAmount) = IPerpetual(
                state.openPositions[trader][i]
            ).balanceOf(trader);
            Types.RiskParams storage params = state.perpRiskParams[
                state.openPositions[trader][i]
            ];
            int256 price = SafeCast.toInt256(
                IMarkPriceSource(params.markPriceSource).getMarkPrice()
            );

            netPositionValue += paperAmount.decimalMul(price) + creditAmount;
            uint256 exposureIncrement = paperAmount.decimalMul(price).abs();
            exposure += exposureIncrement;
            maintenanceMargin +=
                (exposureIncrement * params.liquidationThreshold) /
                Types.ONE;

            unchecked {
                ++i;
            }
        }
    }
```

So a malicious user can add enough positions so that the gas of liquidation transaction reaches the gas limit of a block and always revert.

## Vulnerability Detail

## Impact
Malicious user can prevent liquidation by openning lots of positions due to gas limit

## Code Snippet
https://github.com/JOJOexchange/smart-contract-EVM/blob/4a95a8e9a6367ae88dc827e29467229cb5bbad4f/contracts/lib/Liquidation.sol#L53-L85
## Tool used

Manual Review

## Recommendation
put a limit on total number of open positions for each user



## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/smart-contract-EVM/commit/fbee64482004062de5168c7e0b81525b2a1c1483

# Issue M-18: In over liquidation, if the liquidatee has USDC-denominated assets for sale, the liquidator can buy the assets with USDC to avoid paying USDC to the liquidatee 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/369 

## Found by 
cccz, monrel
## Summary
In over liquidation, if the liquidatee has USDC-denominated assets for sale, the liquidator can buy the assets with USDC to avoid paying USDC to the liquidatee
## Vulnerability Detail
In JUSDBank contract, if the liquidator wants to liquidate more collateral than the borrowings of the liquidatee, the liquidator can pay additional USDC to get the liquidatee's collateral. 
```solidity
        } else {
            //            actualJUSD = actualCollateral * priceOff
            //            = JUSDBorrowed * priceOff / priceOff * (1-insuranceFeeRate)
            //            = JUSDBorrowed / (1-insuranceFeeRate)
            //            insuranceFee = actualJUSD * insuranceFeeRate
            //            = actualCollateral * priceOff * insuranceFeeRate
            //            = JUSDBorrowed * insuranceFeeRate / (1- insuranceFeeRate)
            liquidateData.actualCollateral = JUSDBorrowed
                .decimalDiv(priceOff)
                .decimalDiv(JOJOConstant.ONE - reserve.insuranceFeeRate);
            liquidateData.insuranceFee = JUSDBorrowed
                .decimalMul(reserve.insuranceFeeRate)
                .decimalDiv(JOJOConstant.ONE - reserve.insuranceFeeRate);
            liquidateData.actualLiquidatedT0 = liquidatedInfo.t0BorrowBalance;
            liquidateData.actualLiquidated = JUSDBorrowed;
        }

        liquidateData.liquidatedRemainUSDC = (amount -
            liquidateData.actualCollateral).decimalMul(price);
```
The liquidator needs to pay USDC in the callback and the JUSDBank contract will require the final USDC balance of the liquidatee to increase.
```solidity
        require(
            IERC20(primaryAsset).balanceOf(liquidated) -
                primaryLiquidatedAmount >=
                liquidateData.liquidatedRemainUSDC,
            JUSDErrors.LIQUIDATED_AMOUNT_NOT_ENOUGH
        );
```
If the liquidatee has USDC-denominated assets for sale, the liquidator can purchase the assets with USDC in the callback, so that the liquidatee's USDC balance will increase and the liquidator will not need to send USDC to the liquidatee to pass the check in the JUSDBank contract.
## Impact
In case of over liquidation, the liquidator does not need to pay additional USDC to the liquidatee
## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L188-L204

## Tool used

Manual Review

## Recommendation
Consider banning over liquidation



## Discussion

**JoscelynFarr**

That is in our consideration, if the liquidation triggered, there is a possibility for liquidator to liquidate all collaterals, and the remain collateral will return by the USDC to liquidatee

**JoscelynFarr**

In fact, I don't understand how the attack occurs

**Trumpero**

This issue states that the USDC balance of a liquidated user will be validated as the result of liquidation. However, the liquidator can purchase USDC instead of directly transfer USDC in the callback function (when the liquidated user sells USDC elsewhere). After that, the balance check for liquidation is still fulfilled, but the liquidated user will lose assets.

**hrishibhat**

Additional comment from the Watson:

Assume liquidationPriceOff = 5% and ETH : USDC = 2000 : 1.
Alice's unhealthy position is borrowed for 100000 JUSD, collateral is 60 ETH, meanwhile Alice sells 7 ETH for 14000 USDC in other protocol.
Bob liquidates 60 ETH of Alice's position, Bob needs to pay 100000 JUSD, and 60 * 2000 - 100000 / 0.95 = 14737 USDC. In the JOJOFlashLoan callback, Bob sends 100000 JUSD to the contract and buys the 7 ETH that Alice sold in the other protocol (It increases Alice's USDC balance by 14000), and then Bob just send another 14737-14000=737 USDC to Alice to pass the following check
```
        require(
            IERC20(primaryAsset).balanceOf(liquidated) -
                primaryLiquidatedAmount >=
                liquidateData.liquidatedRemainUSDC,
            JUSDErrors.LIQUIDATED_AMOUNT_NOT_ENOUGH
        );
```

**JoscelynFarr**

fix commit:
https://github.com/JOJOexchange/JUSDV1/commit/5918d68be9b5b021691f768da98df5f712ac6edd

# Issue M-19: FlashLoanLiquidate.JOJOFlashLoan has no slippage control when swapping USDC 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/373 

## Found by 
0x52, 0xStalin, Aymen0909, Bauer, Nyx, T1MOH, cccz, peakbolt, rvierdiiev
## Summary
FlashLoanLiquidate.JOJOFlashLoan has no slippage control when swapping USDC
## Vulnerability Detail
In both GeneralRepay.repayJUSD and FlashLoanRepay.JOJOFlashLoan, the user-supplied minReceive parameter is used for slippage control when swapping USDC. 
```solidity
    function JOJOFlashLoan(
        address asset,
        uint256 amount,
        address to,
        bytes calldata param
    ) external {
        (address approveTarget, address swapTarget, uint256 minReceive, bytes memory data) = abi
            .decode(param, (address, address, uint256, bytes));
        IERC20(asset).approve(approveTarget, amount);
        (bool success, ) = swapTarget.call(data);
        if (success == false) {
            assembly {
                let ptr := mload(0x40)
                let size := returndatasize()
                returndatacopy(ptr, 0, size)
                revert(ptr, size)
            }
        }
        uint256 USDCAmount = IERC20(USDC).balanceOf(address(this));
        require(USDCAmount >= minReceive, "receive amount is too small");
...
    function repayJUSD(
        address asset,
        uint256 amount,
        address to,
        bytes memory param
    ) external {
        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
        uint256 minReceive;
        if (asset != USDC) {
            (address approveTarget, address swapTarget, uint256 minAmount, bytes memory data) = abi
                .decode(param, (address, address, uint256, bytes));
            IERC20(asset).approve(approveTarget, amount);
            (bool success, ) = swapTarget.call(data);
            if (success == false) {
                assembly {
                    let ptr := mload(0x40)
                    let size := returndatasize()
                    returndatacopy(ptr, 0, size)
                    revert(ptr, size)
                }
            }
            minReceive = minAmount;
        }

        uint256 USDCAmount = IERC20(USDC).balanceOf(address(this));
        require(USDCAmount >= minReceive, "receive amount is too small");
```
However, this is not done in FlashLoanLiquidate.JOJOFlashLoan, and the lack of slippage control may expose the user to sandwich attacks when swapping USDC.
## Impact
The lack of slippage control may expose the user to sandwich attacks when swapping USDC.
## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/flashloanImpl/FlashLoanLiquidate.sol#L46-L78

## Tool used

Manual Review

## Recommendation
Consider making FlashLoanLiquidate.JOJOFlashLoan use the minReceive parameter for slippage control when swapping USDC.



## Discussion

**JoscelynFarr**

fix link: https://github.com/JOJOexchange/JUSDV1/commit/b0e7d27cf484d9406a267a1b38ac253113101e8e

# Issue M-20: JUSDBank users can bypass individual collateral borrow limits 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/403 

## Found by 
0x52, Ace-30, GalloDaSballo, J4de, carrotsmuggler, peakbolt
## Summary

JUSDBank imposes individual borrow caps on each collateral. The issue is that this can be bypassed due to the fact that withdraw and borrow use different methods to determine if an account is safe. 

## Vulnerability Detail

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L105-L117

        function borrow(
            uint256 amount,
            address to,
            bool isDepositToJOJO
        ) external override nonReentrant nonFlashLoanReentrant{
            //     t0BorrowedAmount = borrowedAmount /  getT0Rate
            DataTypes.UserInfo storage user = userInfo[msg.sender];
            _borrow(user, isDepositToJOJO, to, amount, msg.sender);
            require(
                _isAccountSafeAfterBorrow(user, getTRate()),
                JUSDErrors.AFTER_BORROW_ACCOUNT_IS_NOT_SAFE
            );
        }

When borrowing the contract calls _isAccountSafeAfterBorrow. This imposes a max borrow on each collateral type that guarantees that the user cannot borrow more than the max for each collateral type. The issues is that withdraw doesn't impose this cap. This allows a user to bypass this cap as shown in the example below.

Example:
Assume WETH and WBTC both have a cap of 10,000 borrow. The user deposits $30,000 WETH and takes a flashloand for $30,000 WBTC. Now they deposit both and borrow 20,000 JUSD. They then withdraw all their WBTC to repay the flashloan and now they have borrowed 20,000 against $30000 in WETH

## Impact

Deposit caps can be easily surpassed creating systematic risk for the system

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L105-L117

## Tool used

Manual Review

## Recommendation

Always use _isAccountSafeAfterBorrow



## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/JUSDV1/commit/611ce809ab1c3c300d888053bea6960ed69ec3c3
https://github.com/JOJOexchange/JUSDV1/commit/0ba5d98aac0e8109f38ebf2382bf84391a7c846b

# Issue M-21: Safety of `updateReserveParam` is not checked, which can bring the protocol to a risky state 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/408 

## Found by 
GalloDaSballo
## Summary

`_liquidationMortgageRate` should be checked against `_initialMortgageRate` before allowing it to be changed

## Vulnerability Detail

Initial Mortgage rate may be close or higher than `_liquidationMortgageRate`, because no check is applied, this can cause the system to trigger liquidations or be in an incorrect state.

Which can break the safety parameters

InitialMortgageRate may be set improperly, which would cause:
- Borrowing too much
- Instant Liquidations

Since the value is not checked against `liquidationMortgageRate`

Additionally, the value is not checked against `liquidationPriceOff` which may cause bad debt

## Impact

All depositors may be incorrectly liquidated

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDOperation.sol#L203-L210

## Tool used

Manual Review

## Recommendation

Check that `_initialMortgageRate` > `_liquidationMortgageRate`



## Discussion

**JoscelynFarr**

fix link:
https://github.com/JOJOexchange/JUSDV1/commit/59439a3d2ededeff00c2645be101b2d70fe1012a

# Issue M-22: GeneralRepay#repayJUSD returns excess USDC to `to` address rather than msg.sender 

Source: https://github.com/sherlock-audit/2023-04-jojo-judging/issues/459 

## Found by 
0x52
## Summary

When using GeneralRepay#repayJUSD to repay a position on JUSDBank, any excess tokens are sent to the `to` address. While this is fine for users that are repaying their own debt this is not good when repaying for another user. Additionally, specifying an excess to repay is basically a requirement when attempting to pay off the entire balance of an account. This combination of factors will make it very likely that funds will be refunded incorrectly.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/flashloanImpl/GeneralRepay.sol#L65-L69

            IERC20(USDC).approve(jusdExchange, borrowBalance);
            IJUSDExchange(jusdExchange).buyJUSD(borrowBalance, address(this));
            IERC20(USDC).safeTransfer(to, USDCAmount - borrowBalance);
            JUSDAmount = borrowBalance;
        }

As seen above, when there is an excess amount of USDC, it is transferred to the `to` address which is the recipient of the repay. When to != msg.sender all excess will be sent to the recipient of the repay rather than being refunded to the caller.

## Impact

Refund is sent to the wrong address if to != msg.sender

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/flashloanImpl/GeneralRepay.sol#L32-L73

## Tool used

Manual Review

## Recommendation

Either send the excess back to the caller or allow them to specify where the refund goes



## Discussion

**JoscelynFarr**

fix link: 
https://github.com/JOJOexchange/JUSDV1/commit/7382ce40dd54f0a396fb5d3f13ab3cfede0493e2

