Little Brick Blackbird

high

# Proposer can drain all Telcoin

## Summary
In TelcoinDistributor.sol, a malicious proposer can drain all Telcoin by exploiting the executeTransaction() function.

## Vulnerability Detail
When a council member proposes a new proposal, they can specify the `totalWithdrawal` amount. This amount is directly transferred from the owner's wallet when calling the function `executeTransaction(uint256 transactionId)`. As there is no limit to cap this withdrawal amount, a malicious proposer can execute a valid proposal, withdrawing the entire balance of the owner. The only limit is the amount of tokens that the owner approved for the contract. According to a response from the protocol team on Discord, this approval amount will likely be less than the maximum for security reasons, but large enough to avoid running a transaction every time.

Protocol team answer in discord for how much the approval amount will be .
 [Likely it will be less than the max for security, but large enough so that a transaction does not need to be run every time](https://discord.com/channels/812037309376495636/1195381710074941601/1195731616547491871). 

The line responsible for the transfer from the owner to the TelcoinDistributor contract:

```solidity
TELCOIN.safeTransferFrom(owner(), address(this), totalWithdrawl);
```
Additionally, a Proof of Concept is provided to illustrate the exact vulnerability described above. Add it in TelcoinDistributor.test.ts and run it using 
```shell
npx hardhat test --grep "should execute a malicious proposal successfully"
```
```typescript
         it('should execute a malicious proposal successfully', async () => {

            
           await telcoin.connect(owner).approve(telcoinDistributor.getAddress(), 100000000000000) // the owner approval based on the test file

           const totalWithdrawl = 10000000000000;
           const destinations = [proposer.address];
           const amounts = [10000000000000];


         //  prposing a malicious proposal that will drain a lot of tokens from the owner balance 
           await expect(telcoinDistributor.connect(proposer).proposeTransaction(totalWithdrawl, destinations, amounts)).to.emit(telcoinDistributor, "TransactionProposed");
           await advanceTime(100);

             await telcoinDistributor.connect(proposer).executeTransaction(1);
            console.log( await telcoin.balanceOf(proposer));// This will be all owner balance 
           
        });
  ```


## Impact
The vulnerability allows a proposer to drain the entire balance owned by the protocol's owner, causing severe damage.

## Code Snippet

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L87-L106
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L143-L175
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L185-L203

## Tool used

Manual Review

## Recommendation
I suggest implementing a maximum limit `maxTotalWithdrawal` for all withdrawals to cap the amount. Additionally, consider adding a setter function for the owner to adjust this limit, either reducing it or increasing it within reasonable bounds. Also, introduce a check for maxTotalWithdrawal in the proposeTransaction() function to ensure proposed transactions adhere to the specified limit.