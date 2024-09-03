| Issue Id | Issue Title                                                                                                                                                                              |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| L-01     | `PhiNFT1155` attempts to refund excess ether sent during art creation but mistakenly sends it to the `PhiFactory` contract rather than the actual caller, causing the function to revert |
| L-02     | `PhiNFT1155` has a payable receive function but no way to recover ether sent to the contract                                                                                             |
| L-03     | In `Cred` the functions `buyShareCred` and `sellShareCred` should have deadlines                                                                                                         |
| NC-01    | `Cred::_createCredInternal` unnecessarily uses the public `buyShareCred` function rather than the internal `_handleTrade` function                                                       |
| NC-02    | `PhiFactory::_processClaim` loads `PhiArt storage art = arts[artId_]`, however `art` is never updated meaning this could be marked `memory` rather than `storage`                        |
| NC-03    | `Cred::createCred` should return `credId` upon completion                                                                                                                                |
| NC-04    | `Cred::_removeCredIdPerAddress` unnecessarily checks `_credIdsPerAddress[sender_].length` rather than the `_credIdsPerAddressArrLength` mapping                                          |
| NC-05    | `PhiNFT1155` inherits and initializes `ReentrancyGuardUpgradeable` but does not use any of its functionality                                                                             |
| NC-06    | The NATSPEC comments for `ICreatorRoyaltiesControl::RoyaltyConfiguration` include information on a `royaltyMintSchedule` param which is no longer present in the code                    |
| NC-07    | Multiple spelling mistakes in the protocol should be fixed                                                                                                                               |

# [L-01] `PhiNFT1155` attempts to refund excess ether sent during art creation but mistakenly sends it to the `PhiFactory` contract rather than the actual caller, causing the function to revert

[Link](https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L151)

If the user sends too large an `artFee` when creating art the `PhiNFT1155` contract attempts to refund the excess:

```solidity
        if ((msg.value - artFee) > 0) {
            _msgSender().safeTransferETH(msg.value - artFee);
        }
```

However in this case `_msgSender()` will be the `PhiFactory` contract, which does not have any method to receive this ether and therefore the function call will revert.

This is shown in the following test added to `PhiFactory.t.sol` here:

```solidity
    function test_Toad_createArtRefundFail() public {
        bytes memory credData = abi.encode(1, owner, "SIGNATURE", 31_337, bytes32(0));
        bytes memory signCreateData = abi.encode(expiresIn, IMAGE_URL, credData);
        bytes32 createMsgHash = keccak256(signCreateData);
        bytes32 createDigest = ECDSA.toEthSignedMessageHash(createMsgHash);
        (uint8 cv, bytes32 cr, bytes32 cs) = vm.sign(claimSignerPrivateKey, createDigest);
        if (cv != 27) cs = cs | bytes32(uint256(1) << 255);
        IPhiFactory.CreateConfig memory config =
            IPhiFactory.CreateConfig(artCreator, receiver, END_TIME, START_TIME, MAX_SUPPLY, MINT_FEE, false);

        // Try createArt sending wrong NFT_ART_CREATE_FEE will revert because PhiNFT1155 tries to send the refund to the PhiFactory which does not have a receive function rather than the actual caller
        vm.expectRevert();
        phiFactory.createArt{ value: NFT_ART_CREATE_FEE + 1 }(signCreateData, abi.encodePacked(cr, cs), config);
    }
```

# [L-02] `PhiNFT1155` has a payable receive function but no way to recover ether sent to the contract

[Link](https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L359)
The contract is able to receive ether but cannot withdraw any ether sent.

# [L-03] In `Cred` the functions `buyShareCred` and `sellShareCred` should have deadlines

[Link](https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L178)
Adding a `deadline` parameter to the `buyShareCred` and `sellShareCred` functions would stop transactions that get stuck in the mempool from being eventually forced through much later when the buyers intention to buy or sell a specific cred may have changed.

# [NC-01] `Cred::_createCredInternal` unnecessarily uses the public `buyShareCred` function rather than the internal `_handleTrade` function

[Link](https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L570)
Using the internal function would save gas and also allow the `buyShareCred` function to be marked `external`, also saving gas for users calling `buyShareCred`

# [NC-02] `PhiFactory::_processClaim` loads `PhiArt storage art = arts[artId_]`, however `art` is never updated meaning this could be marked `memory` rather than `storage`

[Link](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L735)
Changing this would make the code more readable as the reader would understand the `arts[artId_]` mapping will not change. It will also improve gas efficiency.

# [NC-03] `Cred::createCred` should return `credId` upon completion

[Link](https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L232)

# [NC-04] `Cred::_removeCredIdPerAddress` unnecessarily checks `_credIdsPerAddress[sender_].length` rather than the `_credIdsPerAddressArrLength` mapping

[Link](https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L697)
The `_credIdsPerAddressArrLength` exists for this kind of check so failing to use it makes it's existence redudant.

# [NC-05] `PhiNFT1155` inherits and initializes `ReentrancyGuardUpgradeable` but does not use any of its functionality

[Link](https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L108)
Either the contract is missing intended reentrancy guards which should be added, or the inheritance can be removed.

# [NC-06] The NATSPEC comments for `ICreatorRoyaltiesControl::RoyaltyConfiguration` include information on a `royaltyMintSchedule` param which is no longer present in the code

[Link](https://github.com/code-423n4/2024-08-phi/blob/main/src/interfaces/ICreatorRoyaltiesControl.sol#L24)
The stale docs should be updated to remove mention of the `royaltyMintSchedule`

# [NC-07] Multiple spelling mistakes in the protocol should be fixed

[Merke -> Merkle](https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L45)
[initilaized -> initialized](https://github.com/code-423n4/2024-08-phi/blob/main/src/abstract/CreatorRoyaltiesControl.sol#L13)
