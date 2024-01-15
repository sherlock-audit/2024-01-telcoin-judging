Zany Mango Rattlesnake

medium

# Spam proposals

## Summary

An infinite number of active proposals can be initiated by any counselor making it more difficult to challenge them.

## Vulnerability Detail

Members can submit proposals to execute transactions in `TelcoinDistributor`:
```solidity
    function proposeTransaction(
        uint256 totalWithdrawl,
        address[] memory destinations,
        uint256[] memory amounts
    ) external onlyCouncilMember whenNotPaused {
        // Pushing the proposed transaction to the array
        proposedTransactions.push(
            ProposedTransaction({
                totalWithdrawl: totalWithdrawl,
                destinations: destinations,
                amounts: amounts,
                timestamp: uint64(block.timestamp),
                challenged: false,
                executed: false
            })
        );

        // Emitting an event after proposing a transaction
        emit TransactionProposed(proposedTransactions.length - 1, _msgSender());
    }
```
If one of the members disagrees with the transaction that member can challenge the distribution, this way rejecting it.
However, no limits exist on how many active proposals can be submitted. A malicious actor can submit thousands of copy-paste proposals, especially on chains like Polygon where gas cost is meager. Legit members will have a hard time filtering and challenging all these proposals.

## Impact

If evil counselors want to execute some action, they can instantly spam proposals, before their role is revoked. Then white-hat counselors will need to challenge all these proposals fighting a gas war.

## Code Snippet

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L78-L106

## Tool used

Manual Review

## Recommendation

Consider introducing global / per-account limits on proposals. Another useful layer of protection would be to check if the original proposer still has a council member role when executing the transaction.
