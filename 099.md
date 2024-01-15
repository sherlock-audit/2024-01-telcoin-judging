Swift Iris Jaguar

medium

# Sablier stream update in `CouncilMember.sol` can cause loss of funds if the streamed balance is not withdrawn.

## Summary

The vulnerability in the `CouncilMember` contract pertains to the failure to withdraw streamed tokens during a contract stream update, potentially resulting in fund loss for both the contract and the entire council.

## Vulnerability Detail

Sablier Streams facilitate token streaming on a per-second basis, involving a sender who initiates the stream and a receiver who receives the streamed tokens. The receiver can withdraw tokens up to the elapsed seconds from the stream's start. The responsibility to claim streamed tokens lies with the receiver, as stated in the documentation and Sablier stream contracts (read the cancel stream docs [here](https://docs.sablier.com/contracts/v2/guides/stream-management/cancel)) . Once tokens are streamed, the sender cannot withdraw them.

Also sender has the authority to cancel the the stream and claim back the un-streamed amount. But streamed Balance upto the elapsed time can still be claimed by the receiver or person who is approved by the receiver only.

Check the `SablierV2Lockup::cancel()` here 👇:
https://github.com/sablier-labs/v2-core/blob/b0016437ef3cc8606e1100965dd911d7e658b40b/src/abstracts/SablierV2Lockup.sol#L153-L168

Docs for the same could be find here 👇:
https://docs.sablier.com/contracts/v2/guides/stream-management/cancel

As we check from the resources given above, if a stream is canceled only the un-streamed balance will be available for the sender to withdraw. Rest if for the receiver.

The issue arises in the `CouncilMember` contract's functions (`CouncilMember::updateStream(...)`, `CouncilMember::updateID(...)`, and `CouncilMember::updateTarget(...)`) as they do not check whether the entire streamed amount has been withdrawn from the Sablier stream before updating the stream states in the contract. Consequently, if there is an active streamed balance in the Sablier stream, the `CouncilMember` contract will not be able to withdraw it. And now the balance is lying idle in the Sablier stream contract.

However, the previously streamed balance can be reclaimed by adding the old stream back to the `CouncilMember` contract, provided the sender is aware that the streamed balance has not been withdrawn. Nonetheless, complications may arise if modifications are made to the `CouncilMember` contract following the stream update. For instance, the removal of a Council Member could lead to the omitted member not receiving their balance, while the addition of a new member may result in every old member receiving fewer tokens and new members gaining tokens share. This can happen because all update stream functions are handled by the role `GOVERNANCE_COUNCIL_ROLE` in the `CouncilMember` contract. And if it is a multi-sig or governance then it would required a vote to happend in order to perform the new updated. And sponsor confirmed that the multi-sig can be added for this role. Here is the conversation:

**_Question Asked by me:_**
>And last one is, Governance council will be a contract or EOA ( can be multisig). If governance council will be multisig, then how often can it make updates to the contracts?

**_Answer from Sponsor:_**
![image](https://github.com/sherlock-audit/2024-01-telcoin-Aamirusmani1552/assets/90125875/8e438f94-94fa-4fbd-82d9-39796647ae7c)

So if this is the case then new update will be done after some time and a lot of things might happen in that time.

Also the sender's awareness play important role in this. Two scenarios may unfold because of this:
1. The stream has fully distributed its balance, and the sender assumes that the funds have been appropriately allocated to the `CouncilMember` contract.
2. The stream needs to be prematurely canceled for specific reasons, requiring the addition of a new stream.

In both of the scenarios if the sender is unaware then it will be complete loss of tokens.

## Impact

Council members face potential token losses due to the inability to withdraw streamed balances.

## Code Snippet

https://github.com/sherlock-audit/2024-01-telcoin/blob/main/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L229C3-L257C1

## Tool used

- Manual Review

## Recommendation

To mitigate this potential issue, the following actions are advised:

### 1. Implement Sablier Hooks:
Sablier offers essential hooks to address scenarios where the receiver is a contract. These hooks enable the receiver contract to update its state accurately. While these hooks are optional, Sablier strongly recommends their implementation. Of particular relevance in this context is the onStreamCanceled hook, triggered by the Sablier stream contract when the sender cancels the stream. By incorporating this hook in the CouncilMember contract, the receiver can invoke the _retrieve() function upon stream cancellation, ensuring the withdrawal of the entire streamed balance.

**_File: CouncilMember.sol_**

```diff
+ import { ISablierV2LockupRecipient } from "@sablier/v2-core/src/interfaces/hooks/ISablierV2LockupRecipient.sol";

    contract CouncilMember is
        ERC721EnumerableUpgradeable,
        AccessControlEnumerableUpgradeable
+    ISablierV2LockupRecipient
    {

+    function onStreamCanceled(
+        uint256 streamId,
+        uint128, /* senderAmount */
+        uint128 /* recipientAmount */
+    )
+        external
+        pure
+    {
+        _retrieve();
+    }

    }
```  

### 2. Check Stream Depletion in Update Function:
In the stream update function, verify whether the stream is depleted or not. If not, withdraw the streamed tokens before updating the balances. It is crucial to check if the stream is depleted because if the `_retrieve()` function is directly called and the stream has been depleted (all tokens withdrawn by the receiver), invoking `stream.withdrawMax()` will revert. This could lead to a revert in the `_retrieve()` function and potentially cause a denial-of-service (DoS) situation in the stream update function.

**_File: CouncilMember.sol_**
```diff
+    // Syncronize the update process
+    function updateStreamData( IPRBProxy stream_,  address target_, uint256 updateID ) external onlyRole(GOVERNANCE_COUNCIL_ROLE){
+     _checkIfDepleted();    
+     _updateStream(stream_);
+     _updateTarget(target_);
+     _updateID(updateID);
+    }

-    function updateStream(
+    function _updateStream(
        IPRBProxy stream_
-    ) external onlyRole(GOVERNANCE_COUNCIL_ROLE) {
+   ) internal {
        _stream = stream_;
        emit StreamUpdated(_stream);
    }


    /**
     * @notice Update the target address
     * @dev Restricted to the GOVERNANCE_COUNCIL_ROLE.
     * @param target_ New target address.
     */
-    function updateTarget(
+    function _updateTarget(
        address target_
-    ) external onlyRole(GOVERNANCE_COUNCIL_ROLE) {
+   ) internal {
        _target = target_;
        emit TargetUpdated(_target);
    }


    /**
     * @notice Update the ID for a council member
     * @dev Restricted to the GOVERNANCE_COUNCIL_ROLE.
     * @param id_ New ID for the council member.
     */
-       function updateID(uint256 id_) external onlyRole(GOVERNANCE_COUNCIL_ROLE) {
+      function _updateID(uint256 id_) internal {
        _id = id_;
        emit IDUpdated(_id);
      }

+   // assuming IPRBProxy will return results like given below since we have not been provided with PRBProxy in the codebase.
+  // make changes according to the interface to below given function.
+   function _checkIfDepleted() _internal view {
+        (bool success, bytes memory data) = _stream.execute(
+            _target,
+            abi.encodeWithSelector(
+                ISablierV2ProxyTarget.isDepleted.selector,
+                _id
+            )
+        );
        
+      require(success, "Call failed");
+      require(abi.decode(data, (bool)), "Stream is not depleted yet.");
+   }
```
  
**Note**: Make necessary adjustments in the interfaces used.
