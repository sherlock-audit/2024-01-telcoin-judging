Swift Iris Jaguar

high

# `StakingRewardsManager::topUp(...)` Misallocates Funds to `StakingRewards` Contracts

## Summary

The `StakingRewardsManager::topUp(...)` contract exhibits an issue where the specified `StakingRewards` contracts are not topped up at the correct indices, resulting in an incorrect distribution to different contracts.

## Vulnerability Detail

The `StakingRewardsManager::topUp(...)` function is designed to top up multiple `StakingRewards` contracts simultaneously by taking the indices of the contract's addresses in the `StakingRewardsManager::stakingContracts` array. However, the flaw lies in the distribution process: 

```solidity
    function topUp(
        address source,
@>        uint256[] memory indices
    ) external onlyRole(EXECUTOR_ROLE) {
@>        for (uint i = 0; i < indices.length; i++) {
            // get staking contract and config
            StakingRewards staking = stakingContracts[i];
            StakingConfig memory config = stakingConfigs[staking];

            // will revert if block.timestamp <= periodFinish
            staking.setRewardsDuration(config.rewardsDuration);

            // pull tokens from owner of this contract to fund the staking contract
            rewardToken.transferFrom(
                source,
                address(staking),
                config.rewardAmount
            );

            // start periods
            staking.notifyRewardAmount(config.rewardAmount);

            emit ToppedUp(staking, config);
        }
    }
```
GitHub: [[254-278](https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/telx/core/StakingRewardsManager.sol#L254C1-L278C6)]


The rewards are not appropriately distributed to the `StakingRewards` contracts at the specified indices. Instead, they are transferred to the contracts at the loop indices. For instance, if intending to top up contracts at indices `[1, 2]`, the actual top-up occurs at indices `[0, 1]`.

## Impact

The consequence of this vulnerability is that rewards will be distributed to the incorrect staking contract, leading to potential misallocation and unintended outcomes

## Code Snippet

**_Here is a test for PoC:_**

Add the below given test in `StakingRewardsManager.test.ts` File. And use the following command to run the test

```bash
npx hardhat test --grep "TopUp is not done to intended staking rewards contracts"
```

**_TEST:_**

```Javascript
        it("TopUp is not done to intended staking rewards contracts", async function () {
            // add index 2 to indices
            // so topup should be done to index 0 and 2
            indices = [0, 2];

            await rewardToken.connect(deployer).approve(await stakingRewardsManager.getAddress(), tokenAmount * indices.length);
            
            // create 3 staking contracts
            await stakingRewardsManager.createNewStakingRewardsContract(await stakingToken.getAddress(), newStakingConfig);
            await stakingRewardsManager.createNewStakingRewardsContract(await stakingToken.getAddress(), newStakingConfig);
            await stakingRewardsManager.createNewStakingRewardsContract(await stakingToken.getAddress(), newStakingConfig);

            // topup index 0 and 2
            await expect(stakingRewardsManager.connect(deployer).topUp(await deployer.address, indices))
                .to.emit(stakingRewardsManager, "ToppedUp");


            // getting the staking contract at index 0, 1 and 2
            let stakingContract0 = await stakingRewardsManager.stakingContracts(0);
            let stakingContract1 = await stakingRewardsManager.stakingContracts(1);
            let stakingContract2 = await stakingRewardsManager.stakingContracts(2);

            // Staking contract at index 2 should be empty
            expect(await rewardToken.balanceOf(stakingContract2)).to.equal(0);

            // Staking contract at index 0 and 1 should have 100 tokens
            expect(await rewardToken.balanceOf(stakingContract0)).to.equal(100);
            expect(await rewardToken.balanceOf(stakingContract1)).to.equal(100);

        });
```

**_Output:_**

```bash
AAMIR@Victus MINGW64 /d/telcoin-audit/telcoin-audit (main)
$ npx hardhat test --grep "TopUp is not done to intended staking rewards contracts"


  StakingRewards and StakingRewardsFactory
    topUp
      ✔ TopUp is not done to intended staking rewards contracts (112ms)


  1 passing (2s)

```


## Tool used

- Manual Review

## Recommendation

It is recommended to do the following changes:

```diff
    function topUp(
        address source,
        uint256[] memory indices
    ) external onlyRole(EXECUTOR_ROLE) {
        for (uint i = 0; i < indices.length; i++) {
            // get staking contract and config
-            StakingRewards staking = stakingContracts[i];
+           StakingRewards staking = stakingContracts[indices[i]];
            StakingConfig memory config = stakingConfigs[staking];

            // will revert if block.timestamp <= periodFinish
            staking.setRewardsDuration(config.rewardsDuration);

            // pull tokens from owner of this contract to fund the staking contract
            rewardToken.transferFrom(
                source,
                address(staking),
                config.rewardAmount
            );

            // start periods
            staking.notifyRewardAmount(config.rewardAmount);

            emit ToppedUp(staking, config);
        }
    }
```