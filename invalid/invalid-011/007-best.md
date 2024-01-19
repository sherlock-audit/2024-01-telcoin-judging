Exotic Cider Puma

high

# The owner of the TelcoinDistributor contract can freeze transactions/funds

## Summary
An integer overflow using the `challengePeriod` value can be used by the owner that deploys the TelcoinDistributor contract and through the challengePeriod value, he is able to freeze transactions any time he wishes and even get the funds stuck in the contract.

## Vulnerability Detail
### How challengePeriod is defined:
```solidity
// amount of time a proposal can be challenged
uint256 public challengePeriod;
 
constructor(
        IERC20 telcoin,
        uint256 period,
        IERC721 council
    ) Ownable(_msgSender()) {
        // verifies no zero values were used
        require(
            address(telcoin) != address(0) &&
                address(council) != address(0) &&
                period != 0,
            "TelcoinDistributor: cannot intialize to zero"
        );
        // initialize telcoin address
        TELCOIN = telcoin;
        // Initialize challengePeriod duration
        challengePeriod = period;                    // [here]
        // Initialize councilNft address
        councilNft = council;
	[...]
```

```solidity
function setChallengePeriod(uint256 newPeriod) public onlyOwner {
        //update period
        challengePeriod = newPeriod;
        // Emitting an event for new period
        emit ChallengePeriodUpdated(challengePeriod);
    }
```
- The first snippet of code shows that the `challengePeriod` is initialized in the `constructor()` and without any boundary checks.
- Then in the function `setChallengePeriod()` it is only updated by the owner of the new instance of the TelcoinDistributor contract but also without limitations on the new period value.

### challengeTransaction()
```solidity
[...]
require(
            block.timestamp <=
                proposedTransactions[transactionId].timestamp + challengePeriod,
            "TelcoinDistributor: Challenge period has ended"
        );

// Sets the challenged flag of the proposed transaction to true
proposedTransactions[transactionId].challenged = true;
```

Using the challengePeriod, the require() check can evaluate to false any time he wishes. This will end up resulting on transactions that will never be challenged and executed, keeping in mind that  the challenged flag will never be true, the function `executeTransaction()` will revert.

### executeTransaction
```solidity
 // Reverts if the challenge period has not expired
        // [1]
        require(                         
            block.timestamp >
                proposedTransactions[transactionId].timestamp + challengePeriod,
            "TelcoinDistributor: Challenge period has not ended"
        );

        // [2]
        // makes sure the transaction was not challenged
        require(
            !proposedTransactions[transactionId].challenged,
            "TelcoinDistributor: transaction has been challenged"
        );
        // makes sure the transaction was not executed previously
        require(
            !proposedTransactions[transactionId].executed,
            "TelcoinDistributor: transaction has been previously executed"
        );

	// sends out transaction
        batchTelcoin(...)
	[...]
```

At **[1]** that check will be false because of the overflow with challengePeriod.
**[2]** reverts as well because the challengeTransaction() as shown above.



## Impact
- A malicious actor can create new instances of the TelcoinDistributor and have full control when to challenge or execute transactions (Freezing of funds). 
- Since the challengers of proposed transactions are Council Members, this attack will also remove their control over the system.

## Code Snippet

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L55
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L115
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L143
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L210


## Tool used

Manual Review

## Recommendation
Set limitations for the `challengePeriod` value so that  it can not overflow  on the checks highlighted.
