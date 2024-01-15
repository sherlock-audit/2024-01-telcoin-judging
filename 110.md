Melodic Sapphire Goldfish

high

# Guard is wrongly implemented and will freeze the Safe forever once the first transaction hash is vetoed

## Summary

For Safe versions <= 1.4.1: modules get to execute transactions on the corresponding Safe by-passing  guards. This means that transactions coming from the Zodiac module cannot be vetoed.

For Safe versions that include module-guarded transactions: the function that should be used is `checkModuleTransaction()`, not `checkTransaction()`. Transactions coming from the Zodiac module can be vetoed, but the current implementation fails to do so, and, worse, actually all of them get vetoed because the function is not implemented and will revert. This would be the worst scenario, because the Safe and funds in it could get frozen forever.

## Vulnerability Detail

### Safe versions <= 1.4.1 

Safe Modules execute transactions on safes through the `execTransactionFromModule()` function. In Safe versions <= 1.4.1, the guard doesn't get called [here](https://github.com/safe-global/safe-contracts/blob/v1.4.1/contracts/base/ModuleManager.sol#L81). This means the transactions cannot get vetoed. Additionally, for direct transactions to the Safe (not from modules), `checkTransaction()` would revert trying to call `getTransactionHash()` on the caller address if there is at least one transaction vetoed.
 
Note that, even if `execTransactionFromModule()` called the guard, `_msgSender()` in [L63](https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/zodiac/core/SafeGuard.sol#L63) is NOT the zodiac module but the Gnosis safe. The call path is Zodiac Module --> Gnosis Safe --> Guard. The Gnosis safe contract doesn't have a getTransactionHash function, which means the `checkTransaction(...)` would revert anyway. This happens if the function enters the for loop, which will always do after the owner of the contract calls `vetoTransaction(...)` for the first time.

### Safe versions with module-guarded transactions

In the current main implementation of ModuleManager.sol, [the guard is called](https://github.com/safe-global/safe-contracts/blob/ac0eda07aa130b8bb7fdf9c9d3251eaffdb83980/contracts/base/ModuleManager.sol#L64). Note that `checkModuleTransaction()` needs to be implemented in the guard and that the zodiac module address is passed in the last parameter. Be careful of not using msg.sender in the Guard implementation. Otherwise, getTransactionHash will be called on the Safe instead of on the zodiac module.

## Impact

Depends on that Gnosis Safe release. Currently the guard is unable to veto transactions and could potentially freeze the Safe.

## Code Snippet

https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/zodiac/core/SafeGuard.sol#L63

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/zodiac/core/BaseGuard.sol

To the team:
Please be careful when writing tests. Don't just assume that the calls revert the way you think they revert. Tests [here](https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/test/zodiac/SafeGuard.test.ts) pass, but for the wrong reasons. Also make sure to add mock contracts for the whole execution path.

## Tool used

Manual Review

## Recommendation

- Implement the guard correctly according to the Safe version meant to be used.
- Modify [L63](https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/zodiac/core/SafeGuard.sol#L63) so that the Zodiac Module is called instead of the Gnosis Safe.
