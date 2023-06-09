Juntao

medium

# JUSD borrower who is eligible for liquidation may not be liquidated

## Summary
User can borrow JUSD by depositing collateral, when user's account is not safe, it should be liquidated, however, under certain circumstances,  the actual amount of collateral to be liquidated can not be calculated, thus leads to transaction revert.

## Vulnerability Detail
JOJO retrieves collateral (e.g. WBTC) and USDC price data from Chainlink, which returns price in 8 decimals, then converts to WBTC/USDC using division, as USDC is in 6 decimals, the final price needs to be scaled. This is done by:
```solidity
 tokenPrice * JOJOConstant.ONE / decimalsCorrection
```
As confirmed with sponsor team, decimalsCorrect is to make the final price returns the amount of USDC per E18 collateral,  and its value satisfies the following equation:
```solidity
decimalsCorrection =  priceFeeds decimals +  collateral decimals - USDC decimals
```
So decimalsCorrection for WETH is 20 and for WBTC is 10, the same can be viewed in the depolyed contracts: [WETH](0x9c16d4edfc422ed652642b292911be7f91f0dfd7), [WBTC](0x1c57d02febcd84514ec03559cfefea85df078332). It could be seen that as WBTC has smaller decimals (10) than WETH (18), the final price returns for WBTC contains more decimals than that is for WETH, i.e. 16 decimals and 6 decimals respectively.
When liquidating, if the [liquidateAmount](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L400-L402) is larger than [JUSDBorrowed](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L403), the actual amount of collateral to be liquidated is calculated at [L426-L428](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L426-L428):
```solidity
liquidateData.actualCollateral = JUSDBorrowed
            .decimalDiv(priceOff)
            .decimalDiv(JOJOConstant.ONE - reserve.insuranceFeeRate);
```
The problem is, when JUSDBorrowed is small enough (e.g. 0.001u, 1e3), and collateral's priceOff is big enough (e.g. 100_001u, final price is 1e21 + 1), due to precision loss, the actualCollateral will be 0, then the transaction will be reverted due to division error at [L171-L177](https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L171-L177) so user won't be liquidated.
```solidity
require(
// condition: actual liquidate price < max buy price,
// price lower, better
(liquidateData.insuranceFee + liquidateData.actualLiquidated).decimalDiv(liquidateData.actualCollateral)
        <= expectPrice,
    JUSDErrors.LIQUIDATION_PRICE_PROTECTION
);
```
The test codes is as below:
```solidity
contract AuditTest is Test {
    address alice;

    TestERC20 WBTC;
    JUSDBank jusdBank;
    MockChainLink wbtcChainLink;
    
    function setUp() public {
        TestERC20 USDC = new TestERC20("USDC", "USDC", 6);
        WBTC = new TestERC20("BTC", "BTC", 8);

        MockUSDCPrice usdcPrice = new MockUSDCPrice();
        wbtcChainLink = new MockChainLink(200000_00000000); // BTC price is 200k
        JOJOOracleAdaptor wbtcOracle = new JOJOOracleAdaptor(
            address(wbtcChainLink),
            10, // decimalsCorrection
            86400,
            address(usdcPrice)
        );

        JUSD jusd = new JUSD(6);
        jusdBank = new JUSDBank( 
            // maxReservesAmount_
            10,
            // insurance
            address(2),
            address(jusd),
            // jojoDealer
            address(3),
            // maxBorrowAmountPerAccount_
            100000000000,
            // maxBorrowAmount_
            100000000001,
            // borrowFeeRate_
            2e16,
            address(USDC)
        );

        jusdBank.initReserve(
            // token
            address(WBTC),
            // initialMortgageRate
            8e17,
            // maxDepositAmount
            4000e18,
            // maxDepositAmountPerAccount
            2030e18,
            // maxBorrowValue
            100000e6,
            // liquidateMortgageRate
            825e15,
            // liquidationPriceOff
            5e16,
            // insuranceFeeRate
            1e17,
            address(wbtcOracle)
        );

        jusd.mint(type(uint256).max);
        jusd.transfer(address(jusdBank), type(uint256).max);

        alice = address(1);
        WBTC.mint(alice, type(uint256).max);

        vm.startPrank(alice);
        // alice deposits 1 wei BTC
        WBTC.approve(address(jusdBank), type(uint256).max);
        jusdBank.deposit(alice, address(WBTC), 1, alice);

        // alice borrows 0.001u (1e-3)
        jusdBank.borrow(1_000, alice, false);
        vm.stopPrank();
    }

    function testCannotLiquidate() public {
        // BTC price drops to 120k
        wbtcChainLink.setPrice(120000_00000000);

        uint256 maxMintAmount = jusdBank.getDepositMaxMintAmount(alice);
        uint256 t0BorrowBalance = jusdBank.userInfo(alice);
        uint256 JUSDBorrowed = t0BorrowBalance * jusdBank.getTRate() / 1e18;
        // alice's account is not safe now
        assertTrue(JUSDBorrowed > maxMintAmount);

        // try to liquidate but revert
        vm.expectRevert(stdError.divisionError);    // FAIL. Reason: Division or modulo by 0
        jusdBank.liquidate(alice, address(WBTC), address(this), 1, abi.encode("won't go that far"), type(uint256).max);
    }
}

contract MockChainLink is IChainLinkAggregator {
    int256 price;

    constructor(int256 _initialPrice) {
        price = _initialPrice;
    }

    function setPrice(int256 _price) public {
        price = _price;
    }

    function decimals() external view returns (uint8) {
        return 18;
    }

    function description() external view returns (string memory) {
        return "";
    }

    function version() external view returns (uint256) {
        return 1;
    }
    
    function getRoundData(uint80 _roundId)
        external
        view
        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)
    {
        return (1, 1, 1, block.timestamp, 1);
    }

    function latestRoundData()
        external
        view
        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)
    {
        return (1, price, 1, block.timestamp, 1);
    }
}
```
While it may not seem like a big deal as the unliquidatable value in above example is only 1e-3u, please note this value depends on the collateral decimals and price, smaller decimals and higher price can leads to larger unliquidatable amount.

## Impact
As borrower cannot be liquidated when account is not safe, there might be more bad debts and insolvency risk.

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L426-L428
https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBank.sol#L171-L177

## Tool used

Manual Review

## Recommendation

Please consider to modify as below:
```solidity
...
if (amount == liquidatedInfo.depositBalance[collateral]) {
    liquidateData.actualCollateral = amount;
} else {
    liquidateData.actualCollateral = JUSDBorrowed
        .decimalDiv(priceOff)
        .decimalDiv(JOJOConstant.ONE - reserve.insuranceFeeRate);
    if (liquidateData.actualCollateral == 0) {
        liquidateData.actualCollateral = 1
    }
}
...
```


