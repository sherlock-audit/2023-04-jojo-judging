Saeedalipoor01988

medium

# The signature verification process is not complete

## Summary
The signature verification process is not complete, if the order signer is a contract.

## Vulnerability Detail
The contract is using the below code to check the validity of the signature, but the order can get signed from EOA or Contract account, and the below code only works for EOA.

```solidity
Types.Order memory order = orderList[i];
            bytes32 orderHash = EIP712._hashTypedDataV4(
                domainSeparator,
                Trading._structHash(order)
            );
            orderHashList[i] = orderHash;
            address recoverSigner = ECDSA.recover(orderHash, signatureList[i]);
            // requirements
            require(
                recoverSigner == order.signer ||
                    state.operatorRegistry[order.signer][recoverSigner],
                Errors.INVALID_ORDER_SIGNATURE
            );
```

1. if the `order signer` is an EOA, then the `signature` must be a valid ECSDA signature from the `order signer`.
2. if `order signer` is a contract, then `signature` must be checked according to EIP-1271.

## Impact
The signature verification process is not complete, if the order signer is a contract.

## Code Snippet
https://github.com/sherlock-audit/2023-04-jojo/blob/490ea04e6ad6dc6a862b3407b193264b91c6a760/smart-contract-EVM/contracts/impl/JOJOExternal.sol#L139

## Tool used
Manual Review

## Recommendation
Add the below code to make the signature verification process truly

```solidity
  if (Address.isContract(order signer)) {
            require(
                IERC1271(order signer).isValidSignature(orderHash, signatureList[i]) ==
                    ERC1271_MAGICVALUE,
                "failed"
            );
        } else {
            require(
                ECDSA.recover(orderHash, signatureList[i]) == order signer,
                "signature not from order signer"
            );
        }
```