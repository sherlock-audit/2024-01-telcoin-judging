Expert Midnight Rat

high

# Malicious individual council members can challenge ALL transactions without restriction, making them able to prevent ANY transaction from being executed

## Summary

All proposed transactions can be challenged by malicious council members, causing a denial of service in the TelcoinDistributor contract.

## Vulnerability Detail

Malicious CouncilMember NFT holders are able to challenge all of the proposed transactions without limits.

As we can see, the `challengeTransaction()` found in `TelcoinDistributor.sol` simply allows any council member to challenge all the transactions they want:

```solidity
function challengeTransaction( 
        uint256 transactionId
    ) external onlyCouncilMember whenNotPaused {
        // Makes sure the id exists
        require(
            transactionId < proposedTransactions.length,
            "TelcoinDistributor: Invalid index"
        );

        // Reverts if the current time exceeds the sum of the transaction's timestamp and the challenge period
        require(
            block.timestamp <=
                proposedTransactions[transactionId].timestamp + challengePeriod,
            "TelcoinDistributor: Challenge period has ended"
        );

        // Sets the challenged flag of the proposed transaction to true
        proposedTransactions[transactionId].challenged = true;

        // Emits an event with the transaction ID and the challenger's address
        emit TransactionChallenged(transactionId, _msgSender());
    }
```

As per the sponsor’s message in Discord: *“The Council members are semi-trusted. These are people who have been elected and are expected to behave in their own best interest.”.*

## Impact

High. A council member can challenge ALL transactions proposed in the TelcoinDistributor, which breaks the contract’s main purpose and effectively DoS’es its functionality.

## Code Snippet

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L115C5-L136C6

## Tool used

Manual Review

## Recommendation

The contract’s design must be rethinked so that it implements a quorum-based approach to handle proposals. A good starting point would be using [OpenZeppelin’s Governor contract](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/governance/Governor.sol), which implements the mentioned quorum-based approach.
