Smooth Berry Mustang

medium

# StakingRewardsManager::removeStakingRewardsContract should remove a StakingContract also from the StakingRewardsFactory

## Summary
Just to notice:
- Manager = StakingRewardsManager
- Factory = StakingRewardsFactory

Manager#removeStakingRewardsContract is removing a StakingRewards/StakingContract from its-own list/array but and it is not removing that item from the Factory.
It should remove the StakingRewards also from the Factory, exactly like what it does for creating a new StakingRewards contract.
## Vulnerability Detail
Let's see what is doing the Manager for adding a new StakingRewards (i have divided its functionality in 2 parts):
```solidity
function createNewStakingRewardsContract(
        IERC20 stakingToken,
        StakingConfig calldata config
    ) external onlyRole(BUILDER_ROLE) {
        // create the new staking contract
        // new staking will have owner and rewardsDistribution set to address(this)
        StakingRewards staking = StakingRewards(
            address(
                stakingRewardsFactory.createStakingRewards( // <<-------- part 1
                    address(this),
                    IERC20(address(rewardToken)),
                    IERC20(stakingToken)
                )
            )
        );
        //internal call to add new contract
        _addStakingRewardsContract(staking, config); // <<----------- part 2
    }
```

part-1: It calls `createStakingRewards`, lets take a look at it:
```solidity
function createStakingRewards(
        address rewardsDistribution,
        IERC20 rewardsToken,
        IERC20 stakingToken
    ) external onlyOwner returns (StakingRewards) {
        // create contract
        StakingRewards stakingRewards = new StakingRewards(
            rewardsDistribution,
            rewardsToken,
            stakingToken
        );

        // add contract to list
        stakingRewardsContracts.push(stakingRewards);
        //emit values associated
        emit NewStakingRewardsContract(
            getStakingRewardsContractCount() - 1,
            rewardsToken,
            stakingToken,
            StakingRewards(stakingRewards)
        );

        return StakingRewards(stakingRewards);
    }
```
It is deploying a new StakingRewards and then pushes the address of it into `Factory#stakingRewardsContracts` array.

part-2: It calls `_addStakingRewardsContract`:
```solidity
function _addStakingRewardsContract(
        StakingRewards staking,
        StakingConfig calldata config
    ) internal {
        // in order to manage this contract we have to own it
        // staking.acceptOwnership();
        // in order to top up rewards, we have to be rewardsDistribution. this is an onlyOwner function
        staking.setRewardsDistribution(address(this));

        // push staking onto stakingContracts array
        stakingContracts.push(staking);
        // set staking config
        stakingConfigs[staking] = config;
        // mark inclusion in the stakingContracts array
        stakingExists[staking] = true;

        emit StakingAdded(staking, config);
    }
```
This pushes that created StakingRewards (the StakingRewards created by Factory) into `Manager#stakingContracts` array.

Now we have to take a look at `removeStakingRewardsContract`:
```solidity
function removeStakingRewardsContract(
        uint256 i
    ) external onlyRole(BUILDER_ROLE) {
        StakingRewards staking = stakingContracts[i];

        // un-mark this staking contract as included in stakingContracts
        stakingExists[staking] = false;
        // replace the removed staking contract with the last item in the stakingContracts array
        stakingContracts[i] = stakingContracts[stakingContracts.length - 1];
        // pop the last staking contract off the array
        stakingContracts.pop();

        emit StakingRemoved(staking);
    }
```
As you see, this is removing a specified StakingRewards only from the `Manager#stakingContracts`, it means that StakingRewards will remain alive in Factory.
## Impact
Stakers/Users may stake/send their funds/tokens into a deleted/deprecated StakingRewards contract.
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/telx/core/StakingRewardsFactory.sol#L43-L66

https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/telx/core/StakingRewardsManager.sol#L102-L119

https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/telx/core/StakingRewardsManager.sol#L144-L161
## Tool used

Manual Review

## Recommendation
Consider removing StakingRewards also from the Factory.