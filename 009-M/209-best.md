Recumbent Mocha Pheasant

medium

# Random Nonces Can Be Pushed

## Summary

The purpose of a nonce is to uniquely identify a tx/identifier so that the action the nonce is used for can not be replayed , the usual use case for a nonce is that with every nonce used the value is incremented for the next use so that the next time the nonce is assigned it is not the same as before.

## Vulnerability Detail

In the function `vetoTransaction` in `SafeGuard.sol`

```solidity

 function vetoTransaction(
        bytes32 transactionHash,
        uint256 nonce
    ) public onlyOwner {
        // Revert if the transaction has already been vetoed
        if (transactionHashes[transactionHash])
            revert PreviouslyVetoed(transactionHash);
        // Mark the transaction as vetoed
        transactionHashes[transactionHash] = true;
        // Add the nonce of the transaction to the nonces array
        nonces.push(nonce);//AUDIT  - any random nonce can be pushed
    }
```

Even though the function is protected with `onlyOwner` a previously used nonce can be pushed with the transactionHash.
Therefore 2 different transaction hashes might have the same nonce (mistakenly assigned by the trusted party).

Due to this the function `checkTransaction` (which checks if a tx has been vetoed) ->

```solidity
for (uint256 i = 0; i < nonces.length; i++) {
            bytes32 transactionHash = IReality(_msgSender()).getTransactionHash(
                to,
                value,
                data,
                operation,
                nonces[i]
            );
```

This gets the transaction hash (dependent upon the nonce , since the nonce uniquely identifies a hash for let's say a scenario where to , value , data , operation are same for a tx) , therefore if for 2 IDENTICAL tx's with same nonce , this might give erroneous result if one of the tx has been vetoed (then the other also would be marked as vetoed since hash would be same )


## Impact

The tx's can not be uniquely identified if mistakenly equal nonce is assigned for 2 identical tx's

## Code Snippet

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/zodiac/core/SafeGuard.sol#L38

## Tool used

Manual Review

## Recommendation

Make sure the nonce is incremented each time and then assigned.