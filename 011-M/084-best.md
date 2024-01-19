Fun Saffron Bison

high

# Lack of check if proxy executed was successful leading to inaccurate update of rewards

## Summary
`CouncilMember::_retrieve()` is used to retrieve and distribute TELCOIN to council members based on the stream from` _target`. It uses a Sabiler PRBproxy to withdraw tokens to the `CouncilMembers` contract. The problem lies in not checking whether the transaction is successful.

## Vulnerability Detail
The `CouncileMember.sol::retrieve()` calls the `IPRBProxy.sol::execute()` which performs a delegate call to withdraw tokens. In the [IPRBProxy.sol from Etherscan](https://etherscan.io/address/0x638a7aC8315767cEAfc57a6f5e3559454347C3f6#code#F7#L53) execute function's Natspec states that `It returns the data it gets back, and bubbles up any potential revert`. This means that, if the target contract's function being called through execute encounters a revert, that revert is not caught or handled within the execute function itself. Instead, it is allowed to propagate back to the caller of the execute function, and the caller can decide how to handle or react to the revert. 

## Impact
The retrieval and distribution of TELCOIN will be inaccurate hence, causing users to lose their TELCOIN rewards.
## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L270-L279
## Tool used

Manual Review

## Recommendation
1) Handle the `(bytes memory response)` received 
```solidity
bytes memory response = _stream.execute(
    _target,
    abi.encodeWithSelector(
        ISablierV2ProxyTarget.withdrawMax.selector,
        _target,
        _id,
        address(this)
    )
);
// Custom Error helps saves gas
error ExecuteFail();

// Check if the response indicates a revert
if (response.length > 0) {
    // Handle revert
    // You may parse the response to get the revert reason
    // Note: The exact method depends on the implementation of your contracts
    revert ExecuteFail();
}

```