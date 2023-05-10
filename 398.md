GalloDaSballo

medium

# Lack of Redemptions will cause JUSD to trade below peg until liquidations happen

## Summary

Due to lack of redemptions the token may trade below parity (real value) for prolonged time

## Vulnerability Detail

It may be ideal to allow redemptions, perhaps via the JUSDExchange

This means in practice that no-one will prefer borrowing JUSD vs just buying it / swapping for it via the JUSD Exchange.

Because JUSDExchange has no fee, then, it will be very likely for all JUSD to be bought any time there's a need for it, rather than borrowing for it.

This will cause the DAO / Owner of the JUSDExchange to bear the burned of:
- Opportunity Cost (locked capital for no yield)
- Additional Smart Contract Risk (they are using JUSDBank instead of the people purchasing JUSD)

For no benefit

## Impact

Reduced usage, and liquidity of JUSD because of downward price risk that cannot be easily mitigated

## Code Snippet

https://github.com/sherlock-audit/2023-04-jojo/blob/main/JUSDV1/src/Impl/JUSDBankStorage.sol#L51-L52

## Tool used

Manual Review

## Recommendation

Consider adding redemptions or negative interest rates (being paid to borrow)