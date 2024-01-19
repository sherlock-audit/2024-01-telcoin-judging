Virtual Midnight Ladybug

medium

# Missing Range Check in removeStakingRewardsContract Function

## Summary
The removeStakingRewardsContract function in the StakingRewardsManager.sol file lacks a check to ensure that the provided index i is within the bounds of the stakingContracts array. This omission may lead to unexpected memory access issues.
## Vulnerability Detail
The vulnerability lies in the removeStakingRewardsContract function, where the absence of a check on the index may result in accessing memory outside the valid range of the stakingContracts array.
## Impact
This vulnerability could potentially lead to runtime errors, including but not limited to accessing unexpected memory locations, which may compromise the integrity and functionality of the contract.
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/telx/core/StakingRewardsManager.sol#L166-L179
## Tool used

Manual Review

## Recommendation
```solidity
function removeStakingRewardsContract(uint256 i) external onlyRole(BUILDER_ROLE) {
    require(i < stakingContracts.length, "Invalid index");

    StakingRewards staking = stakingContracts[i];

    // Un-mark this staking contract as included in stakingContracts
    stakingExists[staking] = false;

    // Replace the removed staking contract with the last item in the stakingContracts array
    if (i != stakingContracts.length - 1) {
        stakingContracts[i] = stakingContracts[stakingContracts.length - 1];
    }

    // Pop the last staking contract off the array
    stakingContracts.pop();

    emit StakingRemoved(staking);
}
```