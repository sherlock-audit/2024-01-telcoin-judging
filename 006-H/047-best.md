Ambitious Fossilized Puppy

high

# The `CouncilMember` contract DoS due to the `_retrieve` function revert

## Summary

The `_retrieve`function is called before any significant state changes. This function executes the withdrawal from the `_target`, which might be a Sablier stream or another protocol. `SablierV2Lockup` reverts if withdrawable amount equals to `0`.

https://github.com/sablier-labs/v2-core/blob/b0016437ef3cc8606e1100965dd911d7e658b40b/src/abstracts/SablierV2Lockup.sol#L297-L299
https://github.com/sablier-labs/v2-core/blob/b0016437ef3cc8606e1100965dd911d7e658b40b/src/abstracts/SablierV2Lockup.sol#L270-L272

Funds are distributed over time. And even if there are always funds in the protocol for distribution, after calling the `_retrieve` function, a new distribution will not be available until another period of time has passed.

This means that any interaction with the `CouncilMember` contract will be unavailable during this time.

## Vulnerability Detail

1\. If the protocol for distributing funds employs a strategy that permits funds to be released once within a specific timeframe (for instance, 1 day, 1 week, or 1 month), this implies that the `CouncilMember` contract will execute its tasks error-free only once during this period.

2\. The `removeFromOffice` function calls the `_retrieve`function at the beginning to retrieve and distribute any pending TELCOIN for all council members, and transfer token ownership at the end.

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L267-L295

The `_update` function, which is called before each transfer, is overridden and also calls the `_retrieve` function.

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L321-L331

Thus, during the `removeFromOffice` function, the `_retrieve` function will be called twice, which will always result in revert, since after the first distribution of funds, when called again, the withdrawable amount will be `0`.

3\. Also, according to the sponsor's comment, the council members are semi-trusted. A malicious member can prevent others from interacting with the contract. For example:
- Member A wants to claim their allocated amounts of TELCOIN
- Member B fron-runs member's A transaction and call the `retrieve` function
- Member's A transaction reverts because the `_retrieve` function is called again but there are no more withdrawable amount.

## Impact

Denial of Service of the `CouncilMember` contract over a period of time, depending on the fund distribution strategy. The `removeFromOffice` function always fails, leading to the necessity to use the `transferFrom` function, which does not call `_withdrawAll`, potentially breaking the state of the contract

## Code Snippet

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L267-L295

## Tool used

Manual Review

## Recommendation

Check amount before executing withdrawal or wrap call in a `try/catch` block. Also consider abandoning the `removeFromOffice` function, use  `transferFrom` instead and move `_withdrawAll` call to  `_update` function.

