Jolly Foggy Woodpecker

high

# runningBalance calculation is wrong in the function _retrieve(CouncilMember contract)

## Summary
 runningBalance calculation is wrong in the function _retrieve(CouncilMember contract)

## Vulnerability Detail
See the function _retrieve,
1. Let assume totalsupply = 11 and  initialBalance = 500e18;
2. After  Executing the withdrawal from the _target, currentBalance = 1500e18.
3. finalBalance = (1500e18 - 500e18) +0 = 1000e18 [as initially  runningBalance is 0].
4. individualBalance = 1000/11 = 90.90 = 90(0.90 will not consider and as this round down)
5. runningBalance = 1000%11 = 0.90; this calculation is wrong because 11 tokenid gets (90*11) = 990 telcoin tokens. So runningBalance = (1000e18-990e18) = 10e18.

## Impact
Wrong calculation leads to council members getting less rewards.

## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L289C31-L289C31
## Tool used

Manual Review

## Recommendation

Calculate like this,  runningBalance = (finalBalance % totalSupply)*totalSupply.
