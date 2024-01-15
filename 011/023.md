Shambolic Sage Chicken

medium

# Owner must manually fund CouncilMember contract if everyone attempts to claim

## Summary
Owner will have to deposit some TELCOIN into `CouncilMember` for users to claim their entire rewards. This breaks the devs assumption that the only source of TEL is the `_stream`.

## Vulnerability Detail
Imagine there are 3 NFT holders and `_retrieve()` is called. Assume 100e18 in rewards every time `_stream.execute()` is called.

```js
    function _retrieve() internal {

        uint256 initialBalance = TELCOIN.balanceOf(address(this)); 

        _stream.execute(_target, abi.encodeWithSelector(ISablierV2ProxyTarget.withdrawMax.selector, _target, _id, address(this)));

        uint256 currentBalance = TELCOIN.balanceOf(address(this)); 
        uint256 finalBalance = (currentBalance - initialBalance) + runningBalance; 
        uint256 individualBalance = finalBalance / totalSupply(); 
        runningBalance = finalBalance % totalSupply();

        for (uint i = 0; i < balances.length; i++) {
            balances[i] += individualBalance; 
        }
    }
```

`initialBalance`: 0
`currentBalance`: 100,000,000,000,000,000,000
`finalBalance`: 100,000,000,000,000,000,000
`individualBalance`: 33,333,333,333,333,333,333
`runningBalance`: 1
balances increased: 33,333,333,333,333,333,333 for each person, totaling 99,999,999,999,999,999,999

Then all 3 people claim their total 99,999,999,999,999,999,999, making TELCOIN balance go from 100e18 to 1. And `_retrieve()` gets called again.

`initialBalance`: 1
`currentBalance`: 100,000,000,000,000,000,001
`finalBalance`: 100,000,000,000,000,000,002
`individualBalance`: 33,333,333,333,333,333,334
`runningBalance`: 0
balances increased: 33,333,333,333,333,333,334 for each person, totaling 100,000,000,000,000,000,002

Then all 3 people want to claim their total of 100,000,000,000,000,000,002, but contract only has 100,000,000,000,000,000,001.
If the last person goes to claim their entire amount, they will be 1 wei short, and it will revert when they call `claim()` for their entire balance amount. They will have to claim 1 wei less, and will need to wait for the owner to manually fund the `CouncilMember` contract to be able to get their last wei.

## Impact
* Owner will have to deposit some TELCOIN for user to claim their entire reward.
* Because `removeFromOffice()` calls `_withdrawAll()` which attempts to transfer the entire balance of a user, if the other two people have already claimed, then this transaction will revert and not allow the person to be removed from office until the owner manually deposits more funds into `CouncilMember`.

## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L267-#L295

## Tool used
Manual Review

## Recommendation
* Consider removing `runningBalance`, which would ensure the contract always has enough funds via pessimistic accounting of user balances.
* Consider pre-funding `CouncilMember` with some dust or 1e18 TELCOIN on deployment, so it takes care of any excess dust that user balances can accumulate via the `runningBalance` issue.
* Find a better or more precise way to determine leftovers so that the `CouncilMember` contract's TELCOIN balance can never be less than the sum of the user balances.