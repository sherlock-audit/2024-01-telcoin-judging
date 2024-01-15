Melted Pistachio Dalmatian

medium

# `CouncilMember` does not implements `supportsInterface()` correctly.

## Summary
The `CouncilMember` contract inherits from `ERC721EnumerableUpgradeable` but doesn't override the `supportsInterface` method correctly.

## Vulnerability Detail
As we can see from the `CouncilMember.supportsInterface()` code below, it returns `true` when queried about supporting the extension interface (`ERC721EnumerableUpgradeable`), but fails to return `true` when queried about the base interface (`IERC721`) or the `IERC165` interface, both of which it actually supports.

```solidity
    function supportsInterface(
        bytes4 interfaceId
    )
        public
        pure
        override(
            AccessControlEnumerableUpgradeable,
            ERC721EnumerableUpgradeable
        )
        returns (bool)
    {
        return
            interfaceId ==
            type(AccessControlEnumerableUpgradeable).interfaceId ||
            interfaceId == type(ERC721EnumerableUpgradeable).interfaceId;
    }
```

The POC code provided below demonstrates that the `CouncilMember` contract does not accurately return `true` when queried about its support for ERC721. To run the POC code, execute it from the `test/sablier` folder.

```javascript
import { expect } from "chai";
import { ethers } from "hardhat";
import { SignerWithAddress } from "@nomicfoundation/hardhat-ethers/signers";
import { CouncilMember, TestTelcoin, TestStream } from "../../typechain-types";

describe("CouncilMember", () => {
    let admin: SignerWithAddress;
    let support: SignerWithAddress;
    let member: SignerWithAddress;
    let holder: SignerWithAddress;
    let councilMember: CouncilMember;
    let telcoin: TestTelcoin;
    let stream: TestStream;

    let target: SignerWithAddress;
    let id: number = 0;
    let governanceRole: string = ethers.keccak256(ethers.toUtf8Bytes("GOVERNANCE_COUNCIL_ROLE"));
    let supportRole: string = ethers.keccak256(ethers.toUtf8Bytes("SUPPORT_ROLE"));

    beforeEach(async () => {
        [admin, support, member, holder, target] = await ethers.getSigners();

        const TestTelcoinFactory = await ethers.getContractFactory("TestTelcoin", admin);
        telcoin = await TestTelcoinFactory.deploy(admin.address);

        const TestStreamFactory = await ethers.getContractFactory("TestStream", admin);
        stream = await TestStreamFactory.deploy(await telcoin.getAddress());

        const CouncilMemberFactory = await ethers.getContractFactory("CouncilMember", admin);
        councilMember = await CouncilMemberFactory.deploy();

        await councilMember.initialize(await telcoin.getAddress(), "Test Council", "TC", await stream.getAddress(), target.address, id);
        await councilMember.grantRole(governanceRole, admin.address);
        await councilMember.grantRole(supportRole, support.address);
    });

    describe("POC", () => {
        describe("Interface implementation", () => {
            it("ERC721 interface", async () => {
                //Check that it does not support ERC721 interface (0x80ac58cd)
                expect(await councilMember.supportsInterface("0x80ac58cd")).to.equal(false);
            });

            it("ERC165 interface", async () => {
                //Check that it does not support ERC165 interface (0x01ffc9a7)
                expect(await councilMember.supportsInterface("0x01ffc9a7")).to.equal(false);
            });
        });
    });
});
```

## Impact
Due to the wrong `supportsInterface()` implementation, if `CouncilMember` contract is queried (either on-chain or off-chain) it will return incorrectly that it doesn't support such ERCs like ERC721 or ERC165. This will affect `CouncilMember` integration with third-party contracts or off-chain services.

## Code Snippet
https://github.com/sherlock-audit/2024-01-telcoin/blob/0954297f4fefac82d45a79c73f3a4b8eb25f10e9/telcoin-audit/contracts/sablier/core/CouncilMember.sol#L146-L161

## Tool used
Manual Review

## Recommendation
Consider calling the parent contract `supportsInterface()` method, as shown below.

```diff
diff --git a/CouncilMember.sol b/CouncilMember.mod.sol
index 2876ee8..06e6de1 100644
--- a/CouncilMember.sol
+++ b/CouncilMember.mod.sol
@@ -157,7 +157,8 @@ contract CouncilMember is
         return
             interfaceId ==
             type(AccessControlEnumerableUpgradeable).interfaceId ||
-            interfaceId == type(ERC721EnumerableUpgradeable).interfaceId;
+            interfaceId == type(ERC721EnumerableUpgradeable).interfaceId ||
+            super.supportsInterface(interfaceId);
     }
 
     /************************************************
```
