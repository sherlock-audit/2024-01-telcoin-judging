Tall Cobalt Ferret

medium

# SafeGuard.checkTransaction may revert with out of gas

## Summary
SafeGuard.checkTransaction may revert with out of gas as it loops through all nonces.
## Vulnerability Detail
`SafeGuard.checkTransaction` function is develop to check gnosis txs upon execution and do not allow some of them. Owner of guard has ability [to veto tx](https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/zodiac/core/SafeGuard.sol#L28-L39). Then he provides tx hash and nonce. This nonce [is then stored into nonces array](https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/zodiac/core/SafeGuard.sol#L38).

When `checkTransaction` is called, then function will loop through all nonces and call `IReality(_msgSender()).getTransactionHash` to construct tx hash and check if it is vetoed.

During the work of contract it's likely that `nonces` array will be growing till the time, when `checkTransaction` will revert with out of gas error.
## Impact
SafeGuard.checkTransaction will always revert. New guard will be needed.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
I guess that after some time if proposal was not executed, then it should not be able to execute it. So after some delay of time you cacn clear nonces as well.

But i am not sure about that, as i don't know exactly how it will be used,