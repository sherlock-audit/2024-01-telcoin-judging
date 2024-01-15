Fun Saffron Bison

high

# Missing fallback() in Safe Guard incase of Safe upgrade causing the Safe to be locked

## Summary
Safe Guards can perform checks both before and after a Safe transaction. However, it overlooked the possibility of a Safe upgrade, which could result in the Safe being locked.

## Vulnerability Detail
The absence of a `fallback()` function in `SafeGuard.sol` poses a potential vulnerability. This function is crucial in scenarios where the expected check method might change, leading to the risk of locking the Safe. Without a fallback function, the contract cannot receive Ether or more importantly handle calls to undefined functions, a limitation that could hinder the upgrade process. 

The decision not to include a fallback function aims to prevent complications during Safe upgrades, where a reverting fallback could inadvertently lock the Safe, especially if there are changes in the expected check method.

## Impact
Any transaction called to the Safe will be denied.
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/zodiac/core/SafeGuard.sol#L13
## Tool used

Manual Review

## Recommendation
Include a fallback function in SafeGuard.sol. [Reference](https://github.com/gnosisguild/zodiac-guard-scope/blob/3c5dcaf3523c0b8da4fa3924f867ade03fb7f469/contracts/ScopeGuard.sol#L173C1-L177C6)
