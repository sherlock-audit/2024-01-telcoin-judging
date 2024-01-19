Cool Raspberry Rook

medium

# Transfer and burn skips revoking previous allowance

## Summary
When a token is transferred, burned, or minted, the allowance is not reset as it is in a typical ERC721.
## Vulnerability Detail
The CouncilMember NFT contract uses a different storage variable to track allowances than ERC721 does.
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L46

Governance can give an allowance to any token ID holder on behalf of anyone.
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L191-L201

When the token is transferred from one address to another, whether triggered by governance or the approver, the allowance is not reset. So, if governance transfers Alice's NFT to Bob, Alice's NFT's approved target is the same for Bob too. This is not inconsistent with typical ERC721 behavior.

Also, not only for transfers but burns should also revoke the allowance. If governance burns the NFT without revoking the allowances then revoking the burnt NFT's allowance is impossible due to the ownerOf() check inside the event
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L200
## Impact
Since allowance can be given by governance to any arbitrary address transferring an nft could cause problems if the previous allowed target can act maliciously. Since this is fully governance controlled I'll label this as medium rather than a high. 
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L191-L201

https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L46

https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L200

## Tool used

Manual Review

## Recommendation
Inside `_update` revoke the current allowance 