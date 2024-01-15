Cool Raspberry Rook

high

# If any NFT except the last index is burnt, minting new NFT's are impossible

## Summary
When an NFT is burned, the total supply decreases, and the new NFT to be minted uses the totalSupply as the tokenId. Since the totalSupply decreases due to burns, the tokenId assigned to the new NFT will already exist, making the function unusable.
## Vulnerability Detail
When an NFT is burned, the totalSupply decreases in the ERC721Enumerable contract and increases in mints.
https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/dee57da57d313b0823699d1c643d3cf461746c7f/contracts/token/ERC721/extensions/ERC721EnumerableUpgradeable.sol#L20-L26

Assume there are Alice, Bob, and Carol, where Alice holds tokenId 0, Bob holds 1, and Carol holds 2. Hence, the total supply is 3.

When Bob's NFT is burned by governance, the totalSupply will be 2.
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L210-L222

When a new NFT is minted by governance, the tokenId assigned to the new NFT will be the totalSupply(), which is 2. However, tokenId 2 is already in use and held by Carol. That means minting new tokens is impossible!

`Coded PoC:`
```solidity
function test_BurnBricks_NewNftsMints() external {
        ds.dontSendTel();

        vm.prank(deployer);
        cm.grantRole(GOVERNANCE_COUNCIL_ROLE, counciler);

        // @dev counciler mints NFT's to the councileeeeers
        vm.startPrank(counciler);
        cm.mint(tapir);
        cm.mint(hippo);
        cm.mint(ape);

        assertEq(cm.ownerOf(0), tapir);
        assertEq(cm.ownerOf(1), hippo);
        assertEq(cm.ownerOf(2), ape);
        assertEq(cm.totalSupply(), 3);

        cm.burn(1, hippo);  
        assertEq(cm.totalSupply(), 2);

        vm.expectRevert();
        cm.mint(elephant);     
    }
```
## Impact
High since new NFT's can't be minted, system is forever blocked.
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L173-L182
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L210-L222
https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/dee57da57d313b0823699d1c643d3cf461746c7f/contracts/token/ERC721/extensions/ERC721EnumerableUpgradeable.sol#L75-L196
## Tool used

Manual Review

## Recommendation
Don't mint new tokens to "totalSupply()" but instead mint to a counter value that is held in storage and updated accordingly. 