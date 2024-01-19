Expert Midnight Rat

high

# Wrong parameter when retrieving causes a complete DoS of the protocol

## Summary

A wrong parameter in the `_retrieve()` prevents the protocol from properly interacting with Sablier, causing a Denial of Service in all functions calling `_retrieve()`.

## Vulnerability Detail

The `CouncilMember` contract is designed to interact with a Sablier stream. As time passes, the Sablier stream will unlock more TELCOIN tokens which will be available to be retrieved from `CouncilMember`.

The `_retrieve()` internal function will be used in order to fetch the rewards from the stream and distribute them among the Council Member NFT holders (snippet reduced for simplicity):

```solidity
// CouncilMember.sol

function _retrieve() internal {
        ...
        // Execute the withdrawal from the _target, which might be a Sablier stream or another protocol
        _stream.execute(
            _target,
            abi.encodeWithSelector(
                ISablierV2ProxyTarget.withdrawMax.selector, 
                _target, 
                _id,
                address(this)
            )
        );

        ...
    }
```

The most important part in `_retrieve()` regarding the vulnerability that we’ll dive into is the  `_stream.execute()` interaction and the params it receives. In order to understand such interaction, we first need understand the importance of the `_stream` and the `_target` variables.

Sablier allows developers to integrate Sablier via [Periphery contracts](https://github.com/sablier-labs/v2-periphery/tree/main), which prevents devs from dealing with the complexity of directly integrating Sablier’s [Core contracts](https://github.com/sablier-labs/v2-core). Telcoin developers have decided to use these periphery contracts. Concretely, the following contracts have been used:

- [ProxyTarget](https://github.com/sablier-labs/v2-periphery/blob/ba3926d2c3e059a230211077087b73afe46acf64/src/SablierV2ProxyTargetApprove.sol) (link points to an older commit because **[the proxy target contracts have now been deprecated from Sablier](https://github.com/sablier-labs/v2-periphery/pull/226)**): stored in the `_target` variable, this contract acts as the target for a PRBProxy contract. It contains all the complex interactions with the underlying stream. Concretely, Telcoin uses the `[withdrawMax()](https://github.com/sablier-labs/v2-periphery/blob/ba3926d2c3e059a230211077087b73afe46acf64/src/abstracts/SablierV2ProxyTarget.sol#L141C5-L143C6)` function in the proxy target to withdraw all the available funds from the stream (as seen in the previous code snippet).
- [PRBProxy](https://github.com/PaulRBerg/prb-proxy/blob/main/src/PRBProxy.sol): stored in the `_stream` variable, this contract acts as a forwarding (non-upgradable) proxy, acting as a smart wallet that enables multiple contract calls within a single transaction.

> NOTE: It is important to understand that the actual lockup linear stream will be deployed as well. The difference is that the Telcoin protocol  will not interact with that contract directly. Instead, the PRBProxy and proxy target contracts will be leveraged to perform such interactions.
> 

Knowing this, we can now move on to explaining Telcoin’s approach to withdrawing the available tokens from the stream. As seen in the code snippet above, the `_retrieve()` function will perform two steps to actually perform a withdraw from the stream:

It will first call the `_stream`'s  `execute()` function (remember `_stream` is a PRBProxy). This function receives a `target` and some `data` as parameter, and performs a delegatecall aiming at the `target`:

```solidity
// https://github.com/PaulRBerg/prb-proxy/blob/main/src/PRBProxy.sol

/// @inheritdoc IPRBProxy
   function execute(address target, bytes calldata data) external payable override returns (bytes memory response) {
        ...

        // Delegate call to the target contract, and handle the response.
        response = _execute(target, data);
    }

    /*//////////////////////////////////////////////////////////////////////////
                          INTERNAL NON-CONSTANT FUNCTIONS
    //////////////////////////////////////////////////////////////////////////*/

    /// @notice Executes a DELEGATECALL to the provided target with the provided data.
    /// @dev Shared logic between the constructor and the `execute` function.
    function _execute(address target, bytes memory data) internal returns (bytes memory response) {
        // Check that the target is a contract.
        if (target.code.length == 0) {
            revert PRBProxy_TargetNotContract(target);
        }

        // Delegate call to the target contract.
        bool success;
        (success, response) = target.delegatecall(data);

        ...
    }
```

In the `_retrieve()` function, the target where the call will be forwarded to is the `_target` parameter, which is a [ProxyTarget](https://github.com/sablier-labs/v2-periphery/blob/ba3926d2c3e059a230211077087b73afe46acf64/src/SablierV2ProxyTargetApprove.sol) contract. Concretely, the delegatecall function that will be triggered in the [ProxyTarget](https://github.com/sablier-labs/v2-periphery/blob/ba3926d2c3e059a230211077087b73afe46acf64/src/SablierV2ProxyTargetApprove.sol) will be `withdrawMax()`:

```solidity
// https://github.com/sablier-labs/v2-periphery/blob/ba3926d2c3e059a230211077087b73afe46acf64/src/abstracts/SablierV2ProxyTarget.sol#L141C5-L143C6

function withdrawMax(ISablierV2Lockup lockup, uint256 streamId, address to) external onlyDelegateCall {
	lockup.withdrawMax(streamId, to);
}
```

As we can see, the `withdrawMax()` function has as parameters the `lockup` stream contract to withdraw from, the `streamId` and the address `to` which will receive the available funds from the stream. The vulnerability lies in the parameters passed when calling the `withdrawMax()` function in `_retrieve()`. As we can see, the first encoded parameter in the `encodeWithSelector()` call after the selector is the `_target`:

```solidity
// CouncilMember.sol

function _retrieve() internal {
        ...
        // Execute the withdrawal from the _target, which might be a Sablier stream or another protocol
        _stream.execute(
            _target,
            abi.encodeWithSelector(
                ISablierV2ProxyTarget.withdrawMax.selector, 
                _target,   // <------- This is incorrect
                _id,
                address(this)
            )
        );

        ...
    }
```

This means that the proxy target’s `withdrawMax()` function will be triggered with the `_target` contract as the `lockup` parameter, which is incorrect. This will make all calls eventually execute `withdrawMax()` on the PRBProxy contract, always reverting.

The parameter needed to perform the `withdrawMax()` call correctly is the actual Sablier lockup contract, which is currently not stored in the `CouncilMember` contract.

The following diagram also summarizes the current wrong interactions for clarity:
![vulnerability](https://github.com/sherlock-audit/2024-01-telcoin-0xadrii/assets/56537955/ec9c02a4-027e-4f8a-be12-82417bcabf59)

## Impact

High. ALL withdrawals from the Sablier stream will revert, effectively causing a DoS in the _retrieve() function. Because the _retrieve() function is called in all the main protocol functions, this vulnerability essentially prevents the protocol from ever functioning correctly.

## Proof of Concept

Because the current Telcoin repo does not include actual tests with the real Sablier contracts (instead, a `TestStream` contract is used, which has led to not unveiling this vulnerability), [[I’ve created a repository](https://github.com/0xadrii/telcoin-proof-of-concept)](https://github.com/0xadrii/telcoin-proof-of-concept) where the poc can be executed (the repository will be public after the audit finishes (on 15 jan. 2024 at 16:00 CET)). The `testPoc()` function  shows how any interaction (in this case, a call to the `mint()` function) will fail because the proper Sablier contracts are used (PRBProxy and proxy target):

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console2} from "forge-std/Test.sol";
import {SablierV2Comptroller} from "@sablier/v2-core/src/SablierV2Comptroller.sol";
import {SablierV2NFTDescriptor} from "@sablier/v2-core/src/SablierV2NFTDescriptor.sol";
import {SablierV2LockupLinear} from "@sablier/v2-core/src/SablierV2LockupLinear.sol";
import {ISablierV2Comptroller} from "@sablier/v2-core/src/interfaces/ISablierV2Comptroller.sol";
import {ISablierV2NFTDescriptor} from "@sablier/v2-core/src/interfaces/ISablierV2NFTDescriptor.sol";
import {ISablierV2LockupLinear} from "@sablier/v2-core/src/interfaces/ISablierV2LockupLinear.sol";

import {CouncilMember, IPRBProxy} from "../src/core/CouncilMember.sol";
import {TestTelcoin} from "./mock/TestTelcoin.sol";
import {MockProxyTarget} from "./mock/MockProxyTarget.sol";
import {PRBProxy} from "./mock/MockPRBProxy.sol";
import {PRBProxyRegistry} from "./mock/MockPRBProxyRegistry.sol";

import {UD60x18} from "@prb/math/src/UD60x18.sol";
import {LockupLinear, Broker, IERC20} from "@sablier/v2-core/src/types/DataTypes.sol";
import {IERC20 as IERC20OZ} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract PocTest is Test {

    ////////////////////////////////////////////////////////////////
    //                        CONSTANTS                           //
    ////////////////////////////////////////////////////////////////

   bytes32 public constant GOVERNANCE_COUNCIL_ROLE =
        keccak256("GOVERNANCE_COUNCIL_ROLE");
    bytes32 public constant SUPPORT_ROLE = keccak256("SUPPORT_ROLE");

    ////////////////////////////////////////////////////////////////
    //                         STORAGE                            //
    ////////////////////////////////////////////////////////////////

    /// @notice Poc Users
    address public sablierAdmin;
    address public user;

    /// @notice Sablier contracts
    SablierV2Comptroller public comptroller;
    SablierV2NFTDescriptor public nftDescriptor;
    SablierV2LockupLinear public lockupLinear;

    /// @notice Telcoin contracts
    PRBProxyRegistry public proxyRegistry;
    PRBProxy public stream;
    MockProxyTarget public target;
    CouncilMember public councilMember;
    TestTelcoin public telcoin;

    function setUp() public {
        // Setup users
        _setupUsers();

        // Deploy token
        telcoin = new TestTelcoin(address(this));

        // Deploy Sablier 
        _deploySablier();

        // Deploy council member
        councilMember = new CouncilMember();

        // Setup stream
        _setupStream();

        // Setup the council member
        _setupCouncilMember();
    }

    function testPoc() public {
      // Step 1: Mint council NFT to user
      councilMember.mint(user);
      assertEq(councilMember.balanceOf(user), 1);

      // Step 2: Forward time 1 days
      vm.warp(block.timestamp + 1 days);
      
      // Step 3: All functions calling _retrieve() (mint(), burn(), removeFromOffice()) will fail
      vm.expectRevert(abi.encodeWithSignature("PRBProxy_ExecutionReverted()")); 
      councilMember.mint(user);
    }

    function _setupUsers() internal {
        sablierAdmin = makeAddr("sablierAdmin");
        user = makeAddr("user");
    }

    function _deploySablier() internal {
        // Deploy protocol
        comptroller = new SablierV2Comptroller(sablierAdmin);
        nftDescriptor = new SablierV2NFTDescriptor();
        lockupLinear = new SablierV2LockupLinear(
            sablierAdmin,
            ISablierV2Comptroller(address(comptroller)),
            ISablierV2NFTDescriptor(address(nftDescriptor))
        );
    }

    function _setupStream() internal {

        // Deploy proxies
        proxyRegistry = new PRBProxyRegistry();
        stream = PRBProxy(payable(address(proxyRegistry.deploy())));
        target = new MockProxyTarget();

        // Setup stream
        LockupLinear.Durations memory durations = LockupLinear.Durations({
            cliff: 0,
            total: 1 weeks
        });

        UD60x18 fee = UD60x18.wrap(0);

        Broker memory broker = Broker({account: address(0), fee: fee});
        LockupLinear.CreateWithDurations memory params = LockupLinear
            .CreateWithDurations({
                sender: address(this),
                recipient: address(stream),
                totalAmount: 100e18,
                asset: IERC20(address(telcoin)),
                cancelable: false,
                transferable: false,
                durations: durations,
                broker: broker
            });

        bytes memory data = abi.encodeWithSelector(target.createWithDurations.selector, address(lockupLinear), params, "");

        // Create the stream through the PRBProxy
        telcoin.approve(address(stream), type(uint256).max);
        bytes memory response = stream.execute(address(target), data);
        assertEq(lockupLinear.ownerOf(1), address(stream));
    }

    function _setupCouncilMember() internal {
      // Initialize
      councilMember.initialize(
            IERC20OZ(address(telcoin)),
            "Test Council",
            "TC",
            IPRBProxy(address(stream)), // stream_
            address(target), // target_
            1, // id_
            address(lockupLinear)
        );

        // Grant roles
        councilMember.grantRole(GOVERNANCE_COUNCIL_ROLE, address(this));
        councilMember.grantRole(SUPPORT_ROLE, address(this));
    }
  
}
```

## Code Snippet

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L275

## Tool used

Manual Review, foundry

## Recommendation

In order to fix the vulnerability, the proper address needs to be passed when calling `withdrawMax()`. 

> Note that the actual stream address is currently NOT stored in `CouncilMember.sol`, so it will need to be stored (my example shows a new `actualStream` variable)
> 

```diff
function _retrieve() internal {
        ...
        // Execute the withdrawal from the _target, which might be a Sablier stream or another protocol
        _stream.execute(
            _target,
            abi.encodeWithSelector(
                ISablierV2ProxyTarget.withdrawMax.selector, 
-                _target, 
+		actualStream
                _id,
                address(this)
            )
        );

        ...
    }
```
