Striped Gingham Elephant

medium

# `removeFromOffice` uses `_transfer` instead of `_safeTransfer`

## Summary

## Vulnerability Detail
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit%2Fcontracts%2Fsablier%2Fcore%2FCouncilMember.sol#L122-L134

```solidity
    function removeFromOffice(
        address from,
        address to,
        uint256 tokenId,
        address rewardRecipient
    ) external onlyRole(GOVERNANCE_COUNCIL_ROLE) {
        // Retrieve and distribute any pending TELCOIN for all council members
        _retrieve();
        // Withdraw all the TELCOIN rewards for the specified token to the rewardRecipient
        _withdrawAll(rewardRecipient, tokenId);
        // Transfer the token (representing the council membership) from one address to another

//@audit- use safeTransfer instead 
        _transfer(from, to, tokenId);
    }
```
In the context of ERC721, the  `transfer` function itself does not revert if the recipient address does not implement the  `onERC721Received`  function. The  `transfer`  function, unlike  `safeTransfer` , does not include the additional checks to prevent potential loss of tokens in cases where the recipient address does not handle ERC721 tokens as expected.

When using the standard  `transfer `  function for ERC721 token transfers, if the recipient address does not implement the ` onERC721Received` function or is not an ERC721 receiver contract, the transfer will still occur, potentially resulting in a loss of tokens if the recipient cannot handle the incoming tokens appropriately.


## Impact
Potential loss of council membership NFT 
## Code Snippet

## Tool used

Manual Review

## Recommendation
employing  safeTransfer  is recommended as a best practice for ERC721 token transfers within contracts