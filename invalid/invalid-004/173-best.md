Kind Mustard Horse

medium

# function recoverERC20FromStaking calls recoverERC20 with wrong parameters

## Summary
Function `recoverERC20FromStaking` calls `recoverERC20` with wrong parameters

## Vulnerability Detail
Function `recoverERC20` expects to recieve `IERC20 tokenAddress`, `uint256 tokenAmount`, `address to` in this order. However the `recoverERC20FromStaking` function calls the `recoverERC20` function with wrongly arranged parameters
## Impact
The code may not function as intended

## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/telx/core/StakingRewardsManager.sol#L216-L237
``` solidity
function recoverERC20FromStaking(
        StakingRewards staking,
        IERC20 tokenAddress,
        uint256 tokenAmount,
        address to
    ) external onlyRole(SUPPORT_ROLE) {
        // grab the tokens from the staking contract
        staking.recoverERC20(to, tokenAddress, tokenAmount);
    }

function recoverERC20(
        IERC20 tokenAddress,
        uint256 tokenAmount,
        address to
    ) external onlyRole(SUPPORT_ROLE) {
        //move funds
        tokenAddress.safeTransfer(to, tokenAmount);
    }
```
## Tool used
Manual Review

## Recommendation
Instead of `staking.recoverERC20(to, tokenAddress, tokenAmount);`, use `staking.recoverERC20(tokenAddress, tokenAmount, to);`
``` solidity
function recoverERC20FromStaking(
        StakingRewards staking,
        IERC20 tokenAddress,
        uint256 tokenAmount,
        address to
    ) external onlyRole(SUPPORT_ROLE) {
        // grab the tokens from the staking contract
        staking.recoverERC20(tokenAddress, tokenAmount, to);
    }
```