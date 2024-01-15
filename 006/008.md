Savory Bronze Panda

medium

# Unhandled return value of transferFrom in `topUp()` in `StakingRewardsManager.sol` can lead to users being denied their rewards

## Summary
Unhandled return value of transferFrom in `topUp()` in `StakingRewardsManager.sol` can lead to users unable to claim their rewards, because `notifyRewardAmount()` will not revert if there is balance left by users who have not claimed their rewards yet.
## Vulnerability Detail
`topUp` uses `transferFrom` which may return false if the transfer did succeed. However the return value of `transferFrom()` is not checked which will lead to the execution of the `notifyRewardAmount()`. In `StakingRewards.sol` the function `notifyRewardAmount()` does the following check:
```solidity
 // Check the balance of the rewards token in this contract
        uint balance = rewardsToken.balanceOf(address(this));
        // Make sure that the new reward rate isn't higher than what the contract can currently pay out
        require(
            rewardRate <= balance / rewardsDuration,
            "Provided reward too high"
        );
```
However if there are still rewards unclaimed by users from a previous stake, the `notifyRewardAmount()` will not revert. As a result the contract will be put in a state in which the reward it has do not satisfy the user's needs and users will be denied their rewards.
Prove of concept:
Consider the following scenario:
---RewardsAmount is set to 100 and RewardsDuration is set to 7 days
1. Alice stakes 1000 tokens
2. The executor calls topUp() with `source` that has enough `rewardToken` for the `transferFrom()` to be successful
3. One week later Alica calls `earned()` which returns 100, but does not claim the reward leaving it in the `StakingRewards` contract. 
4. The executor calls topUp() again but this time `source` does not have enough reward tokens. However the transaction does not revert because the check in `notifyRewardsAmount` returns true as the balance of the contract's rewardToken is equal to the `RewardsAmount`.
5. One week later Alice calls `earned()` which returns 200, but when she tries to call claim, the transaction reverts as `StakingRewards` has less tokens than what Alice tries to get.
 
## Impact
The contract is put in a state in which users cannot get their rewards. Therefore I consider it Medium.
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/telx/core/StakingRewardsManager.sol?plain=1#L267
## Tool used

Manual Review

## Recommendation
Replace the use of `transferFrom()` in line 267 in `StakingRewardsManager.sol` with `IERC20(rewardToken).safeTransferFrom()`