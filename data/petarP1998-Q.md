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
# Informational

## Informational 01: Wrong function documentation

In `RewardControl::depositBatch` it is stated that `reasons` is optional, but this is not quite true, because the function validates does the function have equal number of params as recepients. 

## Informational 02: Adjust documentation in `BondingCurve`

In `BondingCurve` on several places there is missing documentation.

1. `getPrice` and `getPriceData` are missing documentation
2. For both functions `getBuyPriceAfterFee` and `getSellPriceAfterFee` documentation is missing for the param `credId`

## Informational 03: Missing documentation in `PhiNFT1155`

Documentation is missing on several places. I will describe them one by one below.

1. 

2. 