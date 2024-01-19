Melted Pistachio Dalmatian

high

# Funds can be lost when changing stream parameters in `CouncilMember` contract.

## Summary
Because `CouncilMember._retrieve()` isn't called prior to modifying stream parameters via either `CouncilMember.updateID()` or `CouncilMember.updateTarget()`, any funds accrued since the last `_retrieve()` call will be forfeited.

## Vulnerability Detail
For accurate token distribution, it's crucial to call the internal method `CouncilMember._retrieve()` before significant state changes in the contract, such as token minting or claims. Unfortunately, this step is omitted before important methods like `CouncilMember.updateTarget()` or `CouncilMember.updateID()`. Consequently, funds accrued in the stream but not yet withdrawn won't be distributed.

For example, if the stream is switched, necessitating a modification of the `_id` parameter via `CouncilMember.updateID()`, all funds accumulated in the previous stream since the last `_retrieve()` call will become inaccessible.

## Impact
Loss of funds during stream migrations.

## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L236-L256

## Tool used
Manual Review

## Recommendation
Consider calling `CouncilMember._retrieve()` before setting the new values of `_target` and `_id`, so that funds distribution is done before changing stream parameters.
