Old Fossilized Stallion

high

# `CouncilMembers::_retrieve()` loops over an array of `balances` to stream `individualBalance` and as the array size (council members) grow, gas cost expands until it becomes unusable.

## Summary
- An internal function [CouncilMembers::_retrieve()](https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L291-L294) loops over an array `balances` then adds `individualBalance` on every element. Every element represents nft token holders / Council Members so it is expected to increase over time. As it happens, the gas cost increases until it becomes unusable either by impracticality or until it reaches the block gas limit.

- The `CouncilMembers::_retrieve()` is also used in multiple occasions as listed in ***Vulnerability Detail***.

- Here's a quick look of the [code](https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L262-L295).

    ```solidity
        /**
        * @notice Retrieve and distribute TELCOIN to council members based on the stream from _target
        * @dev This function fetches the maximum possible TELCOIN and distributes it equally among all council members.
        * @dev It also updates the running balance to ensure accurate distribution during subsequent calls.
        */
        function _retrieve() internal {
        ...

            // Add the individual balance to each council member's balance
            for (uint i = 0; i < balances.length; i++) {
                balances[i] += individualBalance;
            }
        }
    ```

## Vulnerability Detail
- As the council members increase, the array size increases because that is where the tokenId is stored. As it happens the gas cost increases until the time it is unusable either by impracticality or until it reaches the block gas limit.

- This internal function is "also" used in multiple functions. It is expected that the adverse effect will spread across them too.
    - `CouncilMembers::retrieve()`
    - `CouncilMembers::claim()`
    - `CouncilMembers::removeFromOffice()`
    - `CouncilMembers::mint()`
    - `CouncilMembers::burn()`
    - `CouncilMembers::_update()`

## Impact
- As council members become too large, functions listed above will never be executed.
- The contract will reach an unusable state.

## Code Snippet
- The whole function: [CouncilMembers::_retrieve()](https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L262-L295)
- The looped array: [Lines 291-294](https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L291-L294)

## Tool used

Manual Review

## Recommendation
Use mapping to track balances per tokenId then implement the code around it. 

```diff
--    uint256[] public balances;
++    mapping (uint256 tokenId => uint256 balance) balances;

```