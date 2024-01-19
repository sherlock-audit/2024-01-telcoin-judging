Happy Yellow Wolf

medium

# CouncilMember:_isAuthorized return false for owner address

## Summary
The implementation of the _isAuthorized function does not align with the core functionality described in the protocol documentation. This discrepancy may lead to incorrect outcomes, particularly when the input address is the owner. 

## Vulnerability Detail
The root cause of the vulnerability is that the _isAuthorized function fails to adhere to the core functionality described in the protocol documentation where it specifies _isAuthorized should "@return True if the address is approved or is the owner, false otherwise.". 

```solidity
// File: telcoin-audit/contracts/sablier/core/CouncilMember.sol
...
     * @return True if the address is approved or is the owner, false otherwise. // <= FOUND: _isAuthorized does not return true for the owner address as documented
...
304:    function _isAuthorized(
305:        address,
306:        address spender,
307:        uint256 tokenId
308:    ) internal view override returns (bool) {
309:        return (hasRole(GOVERNANCE_COUNCIL_ROLE, spender) ||
310:            _tokenApproval[tokenId] == spender); // <= FOUND: only checks for `GOVERNANCE_COUNCIL_ROLE` and approval
311:    }
```
However, the implementation in reality returns `false` if the input address is the owner as it only takes into account `GOVERNANCE_COUNCIL_ROLE` and approval checks (line 309), which contradicts the expected behavior outlined in the documentation. As a result, any operation relying on the return value of _isAuthorized may yield incorrect results for the owner address.

## Impact
The severity of this vulnerability is moderate, as it introduces a mismatch between the documented core functionality and the actual implementation of the _isAuthorized function. This discrepancy should constitute a "Core functionality impact" severity.

## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L304

## Tool used
Manual Review

## Recommendation
It is recommended to modify the _isAuthorized function to align with the core functionality described in the protocol documentation. Specifically, adding a check for ownerOf(tokenId) to the return condition will help ensure consistent behavior.