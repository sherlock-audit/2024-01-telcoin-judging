Perfect Stone Weasel

medium

# No zero address validation in `setRewardsDistribution` function

krkba
## Summary

## Vulnerability Detail
The `setRewardsDistribution` function does not validate the input address. 
## Impact
It can be a zero address, which leads to unexpected behavior.
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/telx/abstract/RewardsDistributionRecipient.sol#L42-L47
## Tool used

Manual Review

## Recommendation
The function should check that the address is not the zero address.