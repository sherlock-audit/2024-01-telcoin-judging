Perfect Stone Weasel

medium

# lack of input validation for array lengths in `batchTelcoin()` function

krkba
## Summary

## Vulnerability Detail
When there is a lack of input validation for array lengths, it means the contract does not verify whether the lengths of destinations array and amounts array match before proceeding with execution the function.
## Impact
Mismatched array lengths can potentially exploited by attcker to manipulate the contract behavior,they may attempt to provide invalid or unexpected data, causing the contract to behave in unintended ways.
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L185-L203

## Tool used

Manual Review

## Recommendation
The contract should check whether the lengths of destinations and amounts arrays match before proceeding.