Cool Raspberry Rook

high

# When governance burns an NFT, the claimable balances of other NFT can be mixed

## Summary
Governance can burn any NFT; however, burning the NFT also forgets to update claimable balances. Balances will be changed, and remaining NFT holders' balances will be mixed up differently than they should be
## Vulnerability Detail
When governance mints NFTs for participants, a separate storage variable, balances (a uint256 array), corresponds to the NFT holders' token IDs.
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L173-L182

For example, if there are participants Alice, Bob, and Carol:

Alice will be minted tokenId 0, and balances[0] represents Alice.
Bob will be minted tokenId 1, and balances[1] represents Bob.
Carol will be minted tokenId 2, and balances[2] represents Carol.
The balances array looks like this: [AliceClaimable, BobClaimable, CarolClaimable].

When claiming TEL coins from the contract, the tokenId and the balances index have to be the same.
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L92-L111

Governance also has the ability to burn an NFT. Burning an NFT will swap the last element in balances with the deleted one and pop the array.
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L210-L222

If governance decides to burn Bob's NFT in the above scenario (tokenId 1), the balances mapping will be like this:
balances = [AliceClaimable, CarolClaimable].
Now, the length of the balances array is 2, and Carol's index is now 1 instead of 2.

If Carol decides to claim, the transaction will fail due to an "array out of bounds" error because Carol's tokenId is 2, and the balances index is 1. Thus, Carol can't claim his claimable TEL coins.

If governance mints another NFT for Dennis, then Carol can claim. However, this time Carol's claimable is not actually Carol's but Dennis's claimable.

`Coded PoC:`
```solidity
function test_BurningNFT_MessesBalances() external {
        address newTapir = address(10);

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

        // @dev my test suite sends 100 * 1e2 tokens everytime stream executes
        // so I expect these values for the individuals
        assertEq(3 * 100 * 1e2, cm.balances(0));
        assertEq(2 * 100 * 1e2, cm.balances(1));
        assertEq(0.5 * 100 * 1e2, cm.balances(2));
        
        console.log("Balances0", cm.balances(0));
        console.log("Balances1", cm.balances(1));
        console.log("Balances2", cm.balances(2));

        // @dev Counciler decides to remove hippo
        ds.dontSendTel(); // this is just to get cleaner values
        cm.burn(1, hippo);

        console.log("Balances0", cm.balances(0));
        console.log("Balances1", cm.balances(1));

        vm.stopPrank();

        // @dev now ape, tokenId holder 2, wants to claim its TEL coins
        // we expect a out of bonds error because of the burn popped out the array but
        // didnt sorted the balances mapping.
        vm.startPrank(ape);
        vm.expectRevert();
        cm.claim(1, 100);
        vm.stopPrank();
    }
```
## Impact
Claimable balances will be messed up hence, high.
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L173-L182

https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L92-L111

https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L210-L222
## Tool used

Manual Review

## Recommendation
1- If the nft holders are not going to be too many (not more than 50 let's say) then don't pop. Popping will also require you to switch tokenId's aswell as balances indexes which is not easily doable. The gas needed to loop potential empty slots shouldn't be too much considering the NFTs are not many.

2- Store the users tokenId -> balance index and use it when claiming.
mapping (uint256 => uint256) tokenIdToIndex;
tokenId(5) -> 2 means: tokenId 5 owners claimable tokens are stored in balances[2]