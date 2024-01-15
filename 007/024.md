Joyful Pine Mockingbird

medium

# TelcoinDistributor.sol - Pausing the contract would disable challenging transactions

## Summary
The TelcoinDistributor contract allows for the proposal, execution and challenging(cancellation) of transactions. It also introduces pausing functionality, probably to deal with external integration pausing and as a security measure. This functionality can give unfair advantage to proposers.

## Vulnerability Detail
The functions for executing and creating proposals are correctly safe-guarded with the ``whenNotPaused`` modifier, stopping the creation and execution of proposals. But an unfair advantage is created because the ``challengeTransaction`` function has the ``whenNotPaused`` modifier as well.
Depending the the duration of the pause it is highly possible for proposals created before the pause, either intentionally via front-running or accidentally, to pass their challenge period, giving council members no way to challenge. Thus creating an unfair advantage during the pause and potentially allowing malicious/unfavorable transactions to reach execution. 

## Impact
Unfair advantage during pause, potential stealing of funds from the ``owner()`` due to inability to challenge proposal

## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L115-L136

## Tool used

Manual Review

## Recommendation
Remove the ``whenNotPaused`` modifier from the ``challengeTransaction`` function.
