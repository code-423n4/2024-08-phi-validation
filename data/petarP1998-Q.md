# QA report

# Lows

## Low 01: `RewardControl::depositBatch` can be simplified

Currently in the `RewardControl::depositBatch`, the logic is making not needed check at the beginning and this consumes excess gas. 

```solidity
for (uint256 i = 0; i < numRecipients; i++) {
     expectedTotalValue += amounts[i];
}
``` 

This check can be executed at the end of the function.

### Recommendation

```diff
function depositBatch(
        address[] calldata recipients,
        uint256[] calldata amounts,
        bytes4[] calldata reasons, // this should be optional
        string calldata comment
    )
        external
        payable
    {
        uint256 numRecipients = recipients.length;

        if (numRecipients != amounts.length || numRecipients != reasons.length) {
            revert ArrayLengthMismatch();
        }

        uint256 expectedTotalValue;
-       for (uint256 i = 0; i < numRecipients; i++) {
-           expectedTotalValue += amounts[i];
-       } 

-       if (msg.value != expectedTotalValue) {
-            revert InvalidDeposit();
-       }

        for (uint256 i = 0; i < numRecipients; i++) {
            address recipient = recipients[i];
            uint256 amount = amounts[i];

            if (recipient == address(0)) {
                revert InvalidAddressZero();
            }

            balanceOf[recipient] += amount;
+           expectedTotalValue += amount;

            emit Deposit(msg.sender, recipient, reasons[i], amount, comment);
        }
+       if (msg.value != expectedTotalValue) {
+            revert InvalidDeposit();
+       }
    }
```

## Low 02: Verification type check missing

In the `Claimable` the logic for both `Claimable::signatureClaim` and `Claimable::merkleClaim` is missing the check for verification type of the art. The claim function assumes that the user has chosen the right verification method.

### Recommendation

Add verification type check for both of the methods.

## Low 03: DOS in `_validateAndCalculateBatch`

In `Cred::_validateAndCalculateBatch` there is a check does a `credId` is already part of the list of batch operations. Currently this check is handled by a list of `credId`, where on every index of the array the contract is storing the credId. But this means that on every batch operation with cred id, the list will be searched one by one. 

If the batch operations are 100 this means that the last 10 iterations of the batch list, will iterate through this list 90, 91 .. 99 times to check if cred id is part of the list.

### Recommendation

Use mapping instead of list. The mapping will be the credId to boolean, where the boolean is indicating does the credId was already part of the batch operations. 

## Low 04: Wrong event information

In `CuratorRewardsDistributor::distribute` at the end of the function `RewardsDistributed` event is emitted. The problem is that the third parameter of the event should be `royaltyFee`, but instead of the fee is passed the amount that is sent back to the user.

### Impact

If there is backend system which needs to visualize the data, wrong data will be shown to the user. Also if this is used for some kind of statistics.

### Recommendation

Pass the royaltyFee or if this is the expected data, refactor the name of the parameter to be not royaltyFee.

## Low 05: `setCredContract` is not emitting event

Function `BondingCurve::setCredContract` is not emitting event after setting cred contract. Consider emitting event at the end of the function.

## Low 06: Missing validation in `createArtFromFactory`

In `PhiFactory::createArt` is missing a check is `msg.value` enough to cover create art fee. Otherwise if the value of msg.value is lower than fee during creation of art in `PhiNFT1155` will revert, because `msg.value - artFee` is lower than zero. 

The below block of code in `PhiNFT1155`, will revert anytime due to the missing check in `PhiFactory`
```
if ((msg.value - artFee) > 0) {
   _msgSender().safeTransferETH(msg.value - artFee); 
}

```

### Recommendation

Add this to the begging of `PhiFactory::createArt`:

```diff
function createArt(
        bytes calldata signedData_,
        bytes calldata signature_,
        CreateConfig memory createConfig_
    )
        external
        payable
        nonReentrant
        whenNotPaused
        returns (address)
    {
+       if(msg.value < artCreateFee) { 
+           revert InvalidMintFee(); 
+.      }
        _validateArtCreationSignature(signedData_, signature_); // audit-check replay attack ? 
        (, string memory uri_, bytes memory credData) = abi.decode(signedData_, (uint256, string, bytes)); // add chain id!!!!!
        ERC1155Data memory erc1155Data = _createERC1155Data(artIdCounter, createConfig_, uri_, credData);
        address artAddress = createERC1155Internal(artIdCounter, erc1155Data);
        artIdCounter++;
        return artAddress;
    }
```

## Low 07: In `_validateAndCalculateBatch` missing check for priceLimits_

In `_validateAndCalculateBatch` there is a missing check, does the length of `priceLimits_` equal to `amounts` and `creds` length. 

### Recommendation

Add this block of code to `_validateAndCalculateBatch`

```diff
function _validateAndCalculateBatch(
        uint256[] calldata credIds_,
        uint256[] calldata amounts_,
        uint256[] calldata priceLimits_,
        bool isBuy
    )
        internal
        view
        returns (
            uint256 totalAmount,
            uint256[] memory prices,
            uint256[] memory protocolFees,
            uint256[] memory creatorFees
        )
    {
        uint256 length = credIds_.length;
+       if (length != priceLimits_.length) {
+           revert InvalidArrayLength();
+       }
        
        if (length != amounts_.length) {
            revert InvalidArrayLength();
        }

        if (length == 0) {
            revert EmptyBatchOperation();
        }
```


## Low 08: Logic in `_executeBatchTrade` can be simplified and reduce cost

The second for cylce in `_executeBatchTrade` is depositing the reward to the `PhiReward`. This action can be added to first cycle instead of iterating second time. 

### Recommendation

You can change the code in the following way:

```diff
for (uint256 i = 0; i < credIds_.length; ++i) {
            uint256 credId = credIds_[i];
            uint256 amount = amounts_[i];

            _updateCuratorShareBalance(credId, curator, amount, isBuy);

            if (isBuy) {
                creds[credId].currentSupply += amount;
                lastTradeTimestamp[credId][curator] = block.timestamp;
            } else {
                if (block.timestamp <= lastTradeTimestamp[credId][curator] + SHARE_LOCK_PERIOD) { // audit-check blocking the position
                    revert ShareLockPeriodNotPassed(
                        block.timestamp, lastTradeTimestamp[credId][curator] + SHARE_LOCK_PERIOD
                    );
                }
                creds[credId].currentSupply -= amount;
            }

+            protocolFeeDestination.safeTransferETH(protocolFees[i]);
+            IPhiRewards(phiRewardsAddress).deposit{ value: creatorFees[i] }(
+                creds[credId].creator, bytes4(keccak256("CREATOR_ROYALTY_FEE")), ""
+            );
+            emit Royalty(creds[credId].creator, credId, creatorFees[i]);

+            emit Trade(curator, credId, isBuy, amounts_[i], prices[i], protocolFees[i], creds[credId].currentSupply);

+            creds[credId].latestActiveTimestamp = block.timestamp;
        }

-        for (uint256 i = 0; i < credIds_.length; ++i) {
-            uint256 credId = credIds_[i];
-
-            protocolFeeDestination.safeTransferETH(protocolFees[i]);
-            IPhiRewards(phiRewardsAddress).deposit{ value: creatorFees[i] }(
-                creds[credId].creator, bytes4(keccak256("CREATOR_ROYALTY_FEE")), ""
-            );
-           emit Royalty(creds[credId].creator, credId, creatorFees[i]);

-            emit Trade(curator, credId, isBuy, amounts_[i], prices[i], - protocolFees[i], creds[credId].currentSupply);
-        }

```

## Low 09: PhiNFT1155 can't be upgraded

`PhiNFT1155` is upgradeable contract, but due to the current implementation it can't be updated, because it can be updated only by owner, in the current situation the owner is `PhiFactory` and there is no method which exposes the update function of the `PhiNFT1155`.

### Impact

Even if the admins of the protocol, want to update the `PhiNFT1155` this is impossible in the current implementation, leading to losing the opportunity to upgrade already deployed `PhiNFT1155`. 

### Recommendation

Expose a method in `PhiFactory` which can be used to call the update method of the `PhiNFT1155`.

## Low 10: Limit the number of creds per user

Currently user can have shares of as many as creds as he wants. This can lead to some problems in `Cred::getPositionsForCurator`, because it iterates through all of the shares per user. 

## Recommendation
Consider limiting the number of creds that user can have.

# Informational

## Informational 01: Everyone can buy shares to anyone

Currently the lock period is only 10 minutes. But if the period increase to weeks, months in the future, malicious user can mint single share to someone and lock all of the funds.

## Informational 02: Wrong function documentation

In `RewardControl::depositBatch` it is stated that `reasons` is optional, but this is not quite true, because the function validates does the function have equal number of params as recepients. 

## Informational 03: whenNotPaused not needed

In `Cred::_createCredInternal` the `whenNotPaused` modifier is not needed because it's already added to the `createCred`.

## Informational 04: Adjust documentation in `BondingCurve`

In `BondingCurve` on several places there is missing documentation.

1. `getPrice` and `getPriceData` are missing documentation
2. For both functions `getBuyPriceAfterFee` and `getSellPriceAfterFee` documentation is missing for the param `credId`

## Informational 05: Minter should implement `onERC1155BatchReceived`

If the minter is contract it should implement `onERC1155BatchReceived` otherwise mint events will throw exception.

## Informational 06: `public` functions not used internally could be marked `external`

Cred::version
Cred::sellShareCred
Cred::buyShareCredFor
Cred::batchBuyShareCred
Cred::batchSellShareCred
Cred::createCred
Cred::getCreatorRoyalty
Cred::getCuratorAddresses
Cred::getCuratorAddressesWithAmount
Cred::getRoot
PhiFactory::version
PhiFactory::contractURI
PhiFactory::checkProof
PhiNFT1155::version
PhiNFT1155::supportsInterface
PhiNFT1155::contractURI
PhiNFT1155::uri
PhiNFT1155::uri (overloaded)
PhiNFT1155::getPhiFactoryContract
PhiNFT1155::getTokenIdFromFactoryArtId
PhiNFT1155::getFactoryArtId
PhiNFT1155::getArtDataFromFactory
PhiNFT1155::safeTransferFrom
PhiNFT1155::safeBatchTransferFrom
BondingCurve::getPriceData
BondingCurve::getBuyPriceAfterFee
BondingCurve::getSellPriceAfterFee 

## Informational 07: Define and use `constant` variables instead of using literals

Here’s the reformatted guidance for handling repeated constant literal values:

Create Constant State Variables for Literal Values

**PhiFactory.sol**
Replace `10_000` with a constant state variable:
        
```solidity
        // Define constant state variables
        uint256 private constant MAX_PROTOCOL_FEE = 10_000;
        uint256 private constant MAX_ART_CREATE_FEE = 10_000;
        
        // Use the constant state variables
        if (protocolFee_ > MAX_PROTOCOL_FEE) revert ProtocolFeeTooHigh();
        if (artCreateFee_ > MAX_ART_CREATE_FEE) revert ArtCreateFeeTooHigh();
```

**Claimable.sol**
Replace repeated `msg.data[4:]` calls with constants:
        
```solidity
        // Define constant state variables
        uint256 private constant DECODE_OFFSET = 4;
        
        // Use the constant state variable
        abi.decode(msg.data[DECODE_OFFSET:], (bytes32, bytes32, address, address, address, uint256, uint256, uint256, string, bytes32));
        abi.decode(msg.data[DECODE_OFFSET:], (address, bytes32[], address, uint256, uint256, bytes32, string));
```

## Informational 08: All events is missing `indexed` fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

## Informational 09: In `Cred::_executeBatchTrade` nonReentrant modifier should be first

This is a best-practice to protect against reentrancy in other modifiers.
