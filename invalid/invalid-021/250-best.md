Clever Linen Fox

medium

# the `batchTelcoin` will always fail due to wrong check in require.

## Summary
the `batchTelcoin` will always fail due to wrong check in require.
## Vulnerability Detail
in contract Telcoindistributor.sol the function `batchTelcoin` will always fail because it  contains a require which revert if the initial balance of this contract and balance aren't equal after the token transfer! 

## Impact
the whole function of batchTelcoin are not going to work at all and will not batchtransfer token! cause this check is blocking it.
## Code Snippet

look at this function first it stores the balance of the contract in the `initialbalance` and then it does the transfer process as it suppose to be but suddenly it checks the `initialbalance` is == to balance which is wrong cause balance of contract going to be changed anyway but it reverts and blocks the transfer process.
```solidity 


    function batchTelcoin(
        uint256 totalWithdrawl,
        address[] memory destinations,
        uint256[] memory amounts
    ) internal {
        // stores inital balance
        uint256 initialBalance = TELCOIN.balanceOf(address(this));
        //transfers amounts
        TELCOIN.safeTransferFrom(owner(), address(this), totalWithdrawl);
        for (uint i = 0; i < destinations.length; i++) {
            TELCOIN.safeTransfer(destinations[i], amounts[i]);
        }
        //initial balance is used instead of zero
        //if 0 is used instead stray Telcoin could DNS operations
        require(
            TELCOIN.balanceOf(address(this)) == initialBalance,
            "TelcoinDistributor: must not have leftovers"
        );
    }
    
```
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/protocol/core/TelcoinDistributor.sol#L185
## Tool used

Manual Review

## Recommendation
- the require at the end of the batchTelcoin should be removed or fixed but not checking equality after transfer.
