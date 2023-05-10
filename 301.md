peakbolt

medium

# Cancellation of orders on JOJOExchange does not prevent order matching on-chain


## Summary

Trade orders are cancelled off-chain by deleting order info and signature. However, that does not guarantee order cannot be matched on-chain as the signature is valid till it expires.

Furthermore, order signatures will be public for partially filled orders.




## Vulnerability Detail

During a trade, order signatures are submitted on-chain via `JOJOExternal.approveTrade()`. As long as the signature is valid, not expired and order is not fully filled, any order sender can still proceed to match the order even if the users had cancelled and matching engine deleted the signatures. So deletion does not prevent matching of cancelled orders.

It is understood from the docs that order senders are currently limited to validated addresses, but the intent is to allow anyone to match orders and receive trading fees in return. So there is a risk that order sender will still match cancelled orders for the financial incentives. 

Even though users can set a short expiration to reduce the risk, it does not provide instant cancellation, which is desirable for unforeseen events (e.g. flash crash, significiant market movements, etc).

Furthermore, for partially filled orders, the order signatures are public as they would have been submitted on-chain for the partial fills.

## Impact

In an event of significant market movements (flash crash, depeg events, etc), users might wish to cancel their existing orders and replace with new orders at a higher/lower price. 

However they could suffer trading loss, as order sender could still use the order signatures that were signed beforehand and match the previous orders intentionally/accidentally.



## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/tree/main/smart-contract-EVM/contracts/impl/JOJOExternal.sol#L103
https://github.com/sherlock-audit/2023-04-jojo/tree/main/smart-contract-EVM/contracts/impl/JOJOExternal.sol#L124
https://github.com/sherlock-audit/2023-04-jojo/tree/main/smart-contract-EVM/contracts/impl/JOJOExternal.sol#L139-L143
https://github.com/sherlock-audit/2023-04-jojo/tree/main/smart-contract-EVM/contracts/impl/JOJOExternal.sol#L159-L163

## Tool used
Manual review

## Recommendation

The solutions are simple and yet provide great benefits and assurance to the users. 

Two possible solutions with pros and cons,

1. Allow cancellation of specific order (easiest and highly granular control)
 Use a `mapping(address=>bool)` to store cancellation status of each order using `orderHash` as the key. Verify the cancellation status in `approveTrade()`.


2. Allow cancellation of all orders (more gas-optimized but less granular control)
Keep track of a `batchId` on-chain for each user using a `mapping(address=>uint32)` and then include it as part of Order signature. My suggestion is to put the `batchId` in Order struct by reducing expiration to uint32 and use the saved space to store `batchId` as uint32. 
On verification of the order signature, check that the user's current `batchId` on-chain matches the `batchId` in the order signature.
So a `cancelAll()` function can be added to increment the `batchId` when the user wish to cancel all previous orders (have smaller `batchId`). This is similar to OpenSea's cancel all listings and offers.



