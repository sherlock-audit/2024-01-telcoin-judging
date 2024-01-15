Lucky Fossilized Chicken

high

# Anyone can call vetoTransaction function as "onlyOwner" is not implemented correctly

## Summary
Anyone can call vetoTransaction function as "onlyOwner" is not implemented correctly

## Vulnerability Detail
Checking the package.json of the contract, it shows that the contract uses Openzeppelin's contract version 5 upward:

`  "dependencies": {
    "@openzeppelin/contracts": "^5.0.1",
    "@openzeppelin/contracts-upgradeable": "^5.0.1"
  }`
  
 The BaseGuard contract imported the Ownable.sol contract, consequently, which is version 5. And Ownable.sol inturn imported Context.sol. Here's the link to Ownable.sol:

https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable.sol
 
 Now let's look at the key functions in Ownable.sol relevant here:

`    constructor(address initialOwner) {
        if (initialOwner == address(0)) {
            revert OwnableInvalidOwner(address(0));
        }
        _transferOwnership(initialOwner);
    }

    modifier onlyOwner() {
        _checkOwner();
        _;
    }

    function owner() public view virtual returns (address) {
        return _owner;
    }

 
    function _checkOwner() internal view virtual {
        if (owner() != _msgSender()) {
            revert OwnableUnauthorizedAccount(_msgSender());
        }
    }

  function _transferOwnership(address newOwner) internal virtual {
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }`

Here's the key functioon in Context.sol:

`   function _msgSender() internal view virtual returns (address) {
        return msg.sender;
    }`
 
In the SafeGuard contract, `_msgSender()` is passed into Ownable and initialized as the owner. However, the vulnerability in using `_msgSender()` as the owner is that `_msgSender()` is a function that only returns msg.sener. That is, the immediate caller. 

What this means is that `onlyOwner` used in `vetoTransaction` function will not revert when anyone calls the function. The `onlyOwner` function as in `Ownable.sol` returns the `initialOwner` set in the `Ownable.sol` constructor. In `SafeGuard.sol`, `initialOwner`in this sense is set to `_msgSender()` - `Ownable(_msgSender())` which calls a function (_msgSender()) in the real sense.

Note that `_msgSender()` is a function in `Context.sol` cited above that merely returns the immediate sender of a transaction. 

The above vulnerability comes from the fact that the contract uses the latest version of Openzeppeliln contracts (version 5 and above) wrongly.

Had it been that the contract uses a version lesser than 5 of Openzeppelin contracts such as v.4.9.5 or lower, this wouldn't have been a vulnerability.

Here's how `Ownable.sol` looks like in v.4.9.5:

`  constructor() {
        _transferOwnership(_msgSender());
    }

 function owner() public view virtual returns (address) {
        return _owner;
    }

  
  function _checkOwner() internal view virtual {
        require(owner() == _msgSender(), "Ownable: caller is not the owner");
    }

 function _transferOwnership(address newOwner) internal virtual {
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }`
    
 See: https://github.com/OpenZeppelin/openzeppelin-contracts/releases/tag/v4.9.5

From the above, ownership was transferred directly to `_msgSender()` in the constructor of `Ownable.sol`. This makes the sender of the contract the owner of the contract.

However, in version 5 used in `SafeGuard.sol`, this is not the case. 


## Impact
Anyone can call `vetoTransaction` function.

## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/zodiac/core/SafeGuard.sol#L20

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/zodiac/core/SafeGuard.sol#L28-L31

## Tool used

Manual Review

## Recommendation
msg.sender should be used instead of _msgSender() in the constructor
