Smooth Berry Mustang

high

# StakingRewardsFactory::createStakingRewards can not be called by StakingRewardsManager

## Summary
I will use the abbreviated names:
- Manager = StakingRewardsManager
- Factory = StakingRewardsFactory

Factory#createStakingRewards is protected by the `onlyOwner`, and also `Manager` is trying to call Factory#createStakingRewards, but Manager contract is never the owner of Factory contract, so all the calls from Manager to Factory#createStakingRewards willl be reverted.

## Vulnerability Detail
We see the Factory is extended from `Ownable` which means the person/address/contract who deploys the Factory will be the `owner` of Factory.
We also see the Manager is trying to call `Factory#createStakingRewards`:
```solidity
function createNewStakingRewardsContract(
        IERC20 stakingToken,
        StakingConfig calldata config
    ) external onlyRole(BUILDER_ROLE) {
        // create the new staking contract
        // new staking will have owner and rewardsDistribution set to address(this)
        StakingRewards staking = StakingRewards(
            address(
                stakingRewardsFactory.createStakingRewards(
                    address(this),
                    IERC20(address(rewardToken)),
                    IERC20(stakingToken)
                )
            )
        );
        //internal call to add new contract
        _addStakingRewardsContract(staking, config);
    }
```
And `createStakingRewards` protected by `onlyOwner`:
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
So we can conclude the `Manager` contract should be the `owner` of `Factory` contract.

But this is not implemented in the code-base and we can never see any code inside the Manager by which the Manager deploys the Factory to be owner of it, or any further function inside Factory which transfers the ownership to Manager.
So the Manager can never be owner of Factory and BUILDER_ROLE will always get a revert when he tries to call Manager#createNewStakingRewardsContract.
## Impact
StakingRewardsManager::createNewStakingRewardsContract is out-of-service and always reverts.
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/telx/core/StakingRewardsFactory.sol#L43-L66

https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/telx/core/StakingRewardsManager.sol#L102-L118
## Tool used

Manual Review

## Recommendation
Consider transferring the ownership of Factory to Manager.