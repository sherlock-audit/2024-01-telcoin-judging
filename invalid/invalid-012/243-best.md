Soft Pistachio Eel

high

# CouncilMember NFT still support `setApprovalForAll` , `safeTransferFrom` `transferFrom` methods

## Summary
CouncilMember NFT still support `setApprovalForAll` , `safeTransferFrom` `transferFrom` methods 

## Vulnerability Detail
The `approve()` function is restricted in CouncilMember NFT contract .
So, the original purpose of this CouncilMember contract is to limit approval and user transfer .

We use `forge inspect` tool to show all methods of `CouncilMember` NFT contract .

```markdown
forge inspect CouncilMember methods
```

And the result shows
```markdown
···
"safeTransferFrom(address,address,uint256)": "42842e0e",
  "safeTransferFrom(address,address,uint256,bytes)": "b88d4fde",
  "setApprovalForAll(address,bool)": "a22cb465",
···
"transferFrom(address,address,uint256)": "23b872dd",
```

These are methods that users can call normally, because `CouncilMember` NFT contract inherit from `ERC721EnumerableUpgradeable`, and no limit. 

## Impact
These function should be limited, and it is not comply project's purpose .

## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L18-L22

```solidity
contract CouncilMember is
    ERC721EnumerableUpgradeable,
    AccessControlEnumerableUpgradeable
{
    using SafeERC20 for IERC20;
```

tranferfrom methods
https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/token/ERC721/ERC721Upgradeable.sol#L160-L185
```solidity
    function transferFrom(address from, address to, uint256 tokenId) public virtual {
        if (to == address(0)) {
            revert ERC721InvalidReceiver(address(0));
        }
        // Setting an "auth" arguments enables the `_isAuthorized` check which verifies that the token exists
        // (from != 0). Therefore, it is not needed to verify that the return value is not 0 here.
        address previousOwner = _update(to, tokenId, _msgSender());
        if (previousOwner != from) {
            revert ERC721IncorrectOwner(from, tokenId, previousOwner);
        }
    }
```

https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/token/ERC721/ERC721Upgradeable.sol#L145C1-L147C6

```solidity
    function setApprovalForAll(address operator, bool approved) public virtual {
        _setApprovalForAll(_msgSender(), operator, approved);
    }
```



## Tool used

Manual Review, VSCode

## Recommendation

Rewrite these methods and add necessary restrictions .
```solidity
@+ function setApprovalForAll(
@+        address to,
@+        uint256 tokenId
@+    )
@+        public
@+        override(ERC721Upgradeable, IERC721)
@+        onlyRole(GOVERNANCE_COUNCIL_ROLE)
@+    {
```


