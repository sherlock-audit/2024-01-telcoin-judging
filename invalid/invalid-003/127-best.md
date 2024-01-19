Bitter Rose Goose

medium

# TelcoinDistributor.sol setChallengePeriod allows to bypass constructor's restriction

## Summary
TelcoinDistributor setChallengePeriod allows to bypass non zero period constructor restriction    

## Vulnerability Detail
TelcoinDistributor.sol on constructor checks for non zero value in the initialization variables, including period variable:  
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L55-L64  

However this non zero check is missing on setChallengePeriod method bypassing constructor restriction, and breaking protocol asumptions  

## Impact
setChallengePeriod function allows to set period to zero value, breaking constructor asumptions.  
Also, a malicious user could take advantage of this to propose and immediately execute and arbitrary transaction making impossible to challenge it    

## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L210-L212  

## Tool used

Manual Review

## Recommendation
It is recommended to add a check for the variable newPeriod for zero value in the setChallengePeriod function like is implemented in the  TelcoinDistributor.sol constructor  

## References
https://github.com/mixbytes/audits_public/blob/master/Lido/1inch%20Rewards%20Manager/README.md#1-no-check-of-the-address-parameter-for-zero