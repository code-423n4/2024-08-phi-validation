# [L-1] share timelock affects existing shares

## Vulnerability details
During a `buy` trade, the users `shareBalance[credId_]` is increamented by `amount` in [`_updateCuratorShareBalance()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L674):
```solidity
        if (isBuy) {
            ---SNIP---
>>          shareBalance[credId_].set(sender_, currentNum + amount_);
        }
```
After this, `lastTradeTimestamp[credId_][curator_]` is updated to current timestamp:
```solidity
lastTradeTimestamp[credId_][curator_] = block.timestamp;
```

Then during [`sell` trade](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L625-L628), the system requires that a certain amount of time (`SHARE_LOCK_PERIOD`) must have passed since the last trade:
```solidity
>>          if (block.timestamp <= lastTradeTimestamp[credId_][curator_] + SHARE_LOCK_PERIOD) {
                revert ShareLockPeriodNotPassed(
                    block.timestamp, lastTradeTimestamp[credId_][curator_] + SHARE_LOCK_PERIOD
                );
            }
```
As seen the function will revert if this condition is not met.

However this design will also impact their `existing shares` which should not be affected.

**Scenario**
- Consider a case where a user already say `1000 shares` in the protocol. It has been `30 minutes` since he last purchased.
- He then makes another purchase worth `200 shares`.
- During this time, his `lastTradeTimestamp` for this `credId_` is set to `block.timestamp`
- When he then tries a sell trade worth `1000 shares`, this will be reverted when in reality, these `shares` were bought earlier.

## Impact
The user is forced to wait for the cooldown period to elapse for `shares` that had passed their cooldown period already.
Plus given the fact that any user can perform buy trades for any other user, this can Dos the user on sell trades

## Recommendation
Consider redesigning the unlock mechanism to only impact the `newly bought shares` instead of affecting all the users shares.


# [L-2] Inadequate array length validation in `_validateAndCalculateBatch()`

## Vulnerability details
[`_validateAndCalculateBatch()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L825-L828) takes the following arrays: 
```solidity
    function _validateAndCalculateBatch(
        uint256[] calldata credIds_,
        uint256[] calldata amounts_,
        uint256[] calldata priceLimits_,
        ---SNIP---
    )
```
However, if only validates two array lengths (`credIds_` & `amounts_`) and omits one, (`priceLimits_`):
```solidity
        uint256 length = credIds_.length;
        if (length != amounts_.length) {  // @audit-info priceLimits_ Not validated
            revert InvalidArrayLength();
        }
```

## Impact
This is an issue because each `credId` should have its own `priceLimit` and therefore, during iteration if the lengths do not match, this could cause issues. 

## Recommendation
Modifiy the check as follows:
```diff
        uint256 length = credIds_.length;
-       if (length != amounts_.length) {
+       if (length != amounts_.length || length != priceLimits_.length) {
            revert InvalidArrayLength();
        }
```

# [L-3] `signature` can be used by anyone in `createArt()`

## Vulnerability details
[`createArt()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L197-L198) requires `signedData_` and `signature_` to complete the operation. These are then [validated](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L207) as follows in [`_validateArtCreationSignature()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L589-L593):
```solidity
    function _validateArtCreationSignature(bytes calldata signedData_, bytes calldata signature_) private view {
        if (_recoverSigner(keccak256(signedData_), signature_) != phiSignerAddress) revert AddressNotSigned();
        (uint256 expiresIn_,,) = abi.decode(signedData_, (uint256, string, bytes));
        if (expiresIn_ <= block.timestamp) revert SignatureExpired();
        // @audit-info Missing sender validation
    }
```
As seen, the function only verifies that the `signature_` has not expired but does not validate that the `msg.sender` is the `sender`. This therefore means that anyone can use any signature to create art.

Consider [`createCred()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L256): This function as well requires `signedData_` and `signature_` but within it, it ensures that the caller is the `sender`:
```solidity
if (sender != _msgSender()) revert Unauthorized();
```
This ensures that only the actual `sender` can use the `signature` in question and not just anyone.

## Impact
Without sender validation, anyone can use any `signature`  and create art as authority is not validated in the function.

## Recommendation
Modify the `signedData_` to accept a parameter, `sender`: This will then be used to verify the caller aduring art creation.


# [L-4] Only revert on insufficient ether

## Vulnerability details
Currently [`batchClaim()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L316) uses the following check to validate ether supplied:
```solidity
        // calc claim fee
        uint256 totalEthValue;
        for (uint256 i = 0; i < encodeDatas_.length; i++) {
            totalEthValue = totalEthValue + ethValue_[i];
        }
>>      if (msg.value != totalEthValue) revert InvalidEthValue();
```
This means that the function will revert when ether supplied is less or greater than what is required.
However this does not sit right for excess ether as the function could be modified to execute on such condition and refund the excess ether thereby elliminating unnecessary reverts.

**Other affected functions**:
1. [`depositBatch()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L69-L76):
This function reverts if the sum of all `amounts` to send to each `recipient` is not equal to `msg.value`:
```solidity
        uint256 expectedTotalValue;
        for (uint256 i = 0; i < numRecipients; i++) {
            expectedTotalValue += amounts[i];
        }

>>      if (msg.value != expectedTotalValue) {
            revert InvalidDeposit();
        }
```
As such, even if the supplied value is greater than required, the function reverts.

## Recommendation
Modify the check to only revert on insufficient ether as follows:

1. `batchClaim()`
```diff
        // calc claim fee
        uint256 totalEthValue;
        for (uint256 i = 0; i < encodeDatas_.length; i++) {
            totalEthValue = totalEthValue + ethValue_[i];
        }
-       if (msg.value != totalEthValue) revert InvalidEthValue();
+       if (msg.value < totalEthValue) revert InvalidEthValue();
+       // @audit Handle refund
+       if ((msg.value - totalEthValue) > 0) {
+           _msgSender().safeTransferETH(msg.value - totalEthValue);
+       }      
```
2. `depositBatch()`
```diff
        uint256 expectedTotalValue;
        for (uint256 i = 0; i < numRecipients; i++) {
            expectedTotalValue += amounts[i];
        }

-       if (msg.value != expectedTotalValue) {
-           revert InvalidDeposit();
-       }
+       if (msg.value < expectedTotalValue) revert InvalidEthValue();
+       // @audit Handle refund
+       if ((msg.value - expectedTotalValue) > 0) {
+           _msgSender().safeTransferETH(msg.value - expectedTotalValue);
+       }
```


# [L-5] `createArt()` has the `whenNotPaused` modifier, but `updateArtSettings()` does not.

## Vulnerability details
The `PhiFactory` contract contains two key functions: `createArt()` for creating new credentials and `updateArtSettings()` for updating existing ones. These functions have different pause controls, which could lead to inconsistent behavior.

The [`createArt()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L204) function includes the `whenNotPaused` modifier, preventing the creation of new credentials when the contract is `paused`.
```solidity
    function createArt(
        ---SNIP---
    )
        external
        payable
        nonReentrant
>>      whenNotPaused
        returns (address)
    {
``` 
However, the [`updateArtSettings()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L215-L257) function lacks this modifier, allowing updates to existing credentials even when the contract is paused. This inconsistency could lead to unexpected behavior and potential security risks.

## Impact
Allowing updates to existing credentials even when the contract is paused could lead to unexpected behavior and potential security risks.

## Recommendation
To ensure consistent behavior and improve security, consider applying the `whenNotPaused` modifier to the `updateArtSettings()` function as well.


# [L-6] `RoyaltyConfiguration` is not validated during updates

## Vulnerability details
The [`_updateRoyalties()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/CreatorRoyaltiesControl.sol#L58-L66) function does not validate the `royaltyBPS` value before setting it.
```solidity
    function _updateRoyalties(uint256 tokenId, RoyaltyConfiguration memory configuration) internal {
        if (configuration.royaltyRecipient == address(0) && configuration.royaltyBPS > 0) {
            revert InvalidRoyaltyRecipient();
        }
        // @audit-info configuration.royaltyBPS is not validated
        royalties[tokenId] = configuration;

        emit UpdatedRoyalties(tokenId, msg.sender, configuration);
    }
```
The contract has [`ROYALTY_BPS_TO_PERCENT`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/CreatorRoyaltiesControl.sol#L11) set as the max at `10_000`. However, this function fails to ensure that `configuration.royaltyBPS` does not fall beyond this value.

`royaltyBPS` is used in `royaltyAmount` calculation as follows:
```solidity
    royaltyAmount = (config.royaltyBPS * salePrice) / ROYALTY_BPS_TO_PERCENT;
```
As seen, if it set beyong `10_000`, this calculation will be flawed as the `royaltyAmount` calculated wold exceed `salePrice`.

`_updateRoyalties()` is called internally in [`PhiNFT1155::updateRoyalties()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L202) which does not verify this value as well.
```solidity
    function updateRoyalties(
        uint256 tokenId_,
        RoyaltyConfiguration memory configuration
    )
        external
        onlyArtCreator(tokenId_)
    {
>>      _updateRoyalties(tokenId_, configuration);
    }
```

---> The `Medium` finding reported in [4naly3er-report.md](https://github.com/code-423n4/2024-08-phi/blob/main/4naly3er-report.md#m-2-fees-can-be-set-to-be-greater-than-100) talks about `fees`. This however, is not about fees. Also, the instances provided in the report does not capture this.

## Recommendation
Add validation to ensure `configuration.royaltyBPS` does not exceed `10_000`:
```diff
    function _updateRoyalties(uint256 tokenId, RoyaltyConfiguration memory configuration) internal {
        if (configuration.royaltyRecipient == address(0) && configuration.royaltyBPS > 0) {
            revert InvalidRoyaltyRecipient();
        }
+       // @audit Perform configuration.royaltyBPS validation
+       if (configuration.royaltyBPS > ROYALTY_BPS_TO_PERCENT) {
+           revert InvalidRoyaltyBPS();
+       }
        royalties[tokenId] = configuration;

        emit UpdatedRoyalties(tokenId, msg.sender, configuration);
    }
```


# [L-7] `getPriceData()` can be refactored

## Vulnerability details
The [`getPriceData()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/curve/BondingCurve.sol#L51-L72) function is used to retrieve the price data. This functions returns the following valuse:  (`price`,  `protocolFee`,  `creatorFee`)
```solidity
function getPriceData(
        uint256 credId_,
        uint256 supply_,
        uint256 amount_,
        bool isSign_
    )
        public
        view
        returns (uint256 price, uint256 protocolFee, uint256 creatorFee)
    {   // @ audit-info Missing credId_ validation
        (uint16 buyShareRoyalty, uint16 sellShareRoyalty) = credContract.getCreatorRoyalty(credId_);

        price = isSign_ ? getPrice(supply_, amount_) : getPrice(supply_ - amount_, amount_);

        protocolFee = _getProtocolFee(price);
        if (supply_ == 0) {
            creatorFee = 0;
            return (price, protocolFee, creatorFee);
        }
        uint16 royaltyRate = isSign_ ? buyShareRoyalty : sellShareRoyalty;
        creatorFee = (price * royaltyRate) / RATIO_BASE;
    }
```
- First, this function does not validate the `credId_` provided which means that if it doesn not exist, the `buyShareRoyalty` and `sellShareRoyalty` retrieved will be all `zeros`.
- Secondly, the function itself contains reduntant lines of code that are already implemented in another function [`_getCreatorFee()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/curve/BondingCurve.sol#L128-L149):
```solidity
    function _getCreatorFee(
        uint256 credId_,
        uint256 supply_,
        uint256 price_,
        bool isSign_
    )
        internal
        view
        returns (uint256 creatorFee)
    {
        if (!credContract.isExist(credId_)) {
            return 0;
        }
        if (supply_ == 0) {
            creatorFee = 0;
        }

        (uint16 buyShareRoyalty, uint16 sellShareRoyalty) = credContract.getCreatorRoyalty(credId_);

        uint16 royaltyRate = isSign_ ? buyShareRoyalty : sellShareRoyalty;
        creatorFee = (price_ * royaltyRate) / RATIO_BASE;
    }
```
As seen above, this function:
1. Validates the provided `credId_`
2. Implements the full functionality that is redundant in `getPriceData()` above.

Therefore, `getPriceData()` need to be refactored.

## Recommendation
Refactor as follows:
```diff
    function getPriceData(
        uint256 credId_,
        uint256 supply_,
        uint256 amount_,
        bool isSign_
    )
        public
        view
        returns (uint256 price, uint256 protocolFee, uint256 creatorFee)
    {   
-       (uint16 buyShareRoyalty, uint16 sellShareRoyalty) = credContract.getCreatorRoyalty(credId_);

        price = isSign_ ? getPrice(supply_, amount_) : getPrice(supply_ - amount_, amount_);

        protocolFee = _getProtocolFee(price);
-       if (supply_ == 0) {
-           creatorFee = 0;
-           return (price, protocolFee, creatorFee);
-       }
-       uint16 royaltyRate = isSign_ ? buyShareRoyalty : sellShareRoyalty;
-       creatorFee = (price * royaltyRate) / RATIO_BASE;
+       creatorFee =  _getCreatorFee(credId_, supply_, price, isSign_);
    }
```

# [L-8] Signatures can be replayed on different chains.

## Vulnerability details
The reported issue on signature replay ([[M-06] createCred() and updateCred() are vulnerable to signature replay attack](https://github.com/code-423n4/2024-08-phi/tree/main/docs/audit)) addresses the scenario where users can use a `signature` and create/update credentials with the same `signedData_` or even `outdated signatures`.

However, it doesn not address `cross-chain repaly`.

The problem is that if the contracts are deployed on different EVM chains and the `sender` account is the same, the same `signature` can be used accross all deployed chains as the `signature` does not contain `block.chainid`

Affected functions:
1. [`createCred()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L234-L235)
2. [`updateCred()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L284-L285)

## Impact

## Recommendation
Add `block.chainid` to the digest of the proof, so each proof is different on each chain.


# [L-9] `endTime` is not properly validated during art creation

## Vulnerability details
[`createArt()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L210) calls [`createERC1155Internal()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L553) which internally calls [`_validateArtCreation()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L585) to perform basic validations of the creation inputs:
```solidity
    function _validateArtCreation(ERC1155Data memory createData_) private view {
        if (arts[createData_.artId].artAddress != address(0)) revert ArtAlreadyCreated();
>>      if (createData_.endTime <= block.timestamp) revert EndTimeInPast(); // @audit-info Incorrect check
        //...
    }
```
However, `createData_.endTime` is incorrectly validated. On revert, the error thrown states that end time is in the past but the operator `<=` used does not enfore this.

## Impact
These use of `<=` operator in `endTime` validation results in the function reverting before `endtime` has passed.

## Recommendation
Modify the check as follows;
```diff
    function _validateArtCreation(ERC1155Data memory createData_) private view {
        if (arts[createData_.artId].artAddress != address(0)) revert ArtAlreadyCreated();
-       if (createData_.endTime <= block.timestamp) revert EndTimeInPast();
+       if (createData_.endTime < block.timestamp) revert EndTimeInPast();
        //...
    }
```

# [L-10] Invalid time range set in `updateArtSettings()`

## Vulnerability details
During art creation, [_validateArtCreation()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L586) requires that `endTime` is not less or equal to `startTime`
```solidity
    function _validateArtCreation(ERC1155Data memory createData_) private view {
        ---SNIP---
>>      if (createData_.endTime <= createData_.startTime) revert EndTimeLessThanOrEqualToStartTime();
    }
```

However, when updating these parameters, the [`updateArtSettings()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L233-L235) function uses an invalid time range for `startTime` and `endTime`.
```solidity
>>      if (endTime_ < startTime_) {
            revert InvalidTimeRange();
        }
```
Here, the check only ensures that `endTime_` is not less that `startTime_` but does not check if these values are equal. As such, an invalid time range may be set.

The [`_validateAndUpdateClaimState()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L710-L711) function checks if the current `block.timestamp` is between `startTime` and `endTime`.
```solidity
        if (block.timestamp < art.startTime) revert ArtNotStarted();
        if (block.timestamp > art.endTime) revert ArtEnded();
```
With `startTime` equal to `endTime`, any claim attempt will revert unless it happens at the exact timestamp, which is highly impractical.

## Impact
Without checking if `endTime_` is equal to  `startTime_`, the whole logic will result in an invalid time range.
The claim period would be effectively non-existent. The art would be available for claiming only at the exact moment when `startTime` equals `endTime`. This would likely result in no successful claims, as the window is too narrow for practical use.

## Recommendation
Modify the check as follows;
```diff
-       if (endTime_ < startTime_) {
+       if (endTime_ <= startTime_) {
            revert InvalidTimeRange();
        }
```

# [L-11] Missing refund of excess ether in `claim()`

## Vulnerability details
Both [`merkleClaim()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L378) and [`signatureClaim()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L343) are `payable` function that pass `msg.value` with the call. Within these functions [`_processClaim()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L737-L741) is called internally with the passed `msg.value` as `etherValue_`.

The function then `refunds` any `excess ether` as follows:
```solidity
        // Handle refund
        uint256 mintFee = getArtMintFee(artId_, quantity_);
        if ((etherValue_ - mintFee) > 0) {
            _msgSender().safeTransferETH(etherValue_ - mintFee);
        }
```
However, in `claim()`, `excess ether` will not be refunded.
This is because, [`merkleClaim()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L282-L283) and [`signatureClaim()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L299-L300) are called within it but with already calculated `mintFee`.
```solidity
    uint256 mintFee = getArtMintFee(artId, quantity_);
>>  this.merkleClaim{ value: mintFee }(proof, claimData, mintArgs, leafPart_);
```
```solidity
    uint256 mintFee = getArtMintFee(artId, quantity_);
>>  this.signatureClaim{ value: mintFee }(signature_, claimData, mintArgs);
```
So, the `excess ether` is not refunded as it is not passed to these functions. These functions will receive `exact` amount of ether required meaning that the `refund logic` in `_processClaim()` will not be triggered as there will be nothing to refund.

Also, the `claim()` function itself does not implement this refund logic and as such, users will have to forfeit these funds.

## Impact
Users will lose their excess ether when calling `claim()` function.

## Recommendation
Update `claim()` to refund any excess payment.
```diff
    uint256 mintFee = getArtMintFee(artId, quantity_);
+   // @audit Handle refund
+   if ((msg.value - mintFee) > 0) {
+       _msgSender().safeTransferETH(msg.value - mintFee);
+   }
    this.merkleClaim{ value: mintFee }(proof, claimData, mintArgs, leafPart_);
```
```diff
    uint256 mintFee = getArtMintFee(artId, quantity_);
+   // @audit Handle refund
+   if ((msg.value - mintFee) > 0) {
+       _msgSender().safeTransferETH(msg.value - mintFee);
+   }
    this.signatureClaim{ value: mintFee }(signature_, claimData, mintArgs);
```

# [L-12]  `RewardControl::deposit()` may overflow

## Vulnerability details
The use of `unchecked` for updating `balances` can lead to `overflow` if the balance approaches the `uint256` limit. This can corrupt balance tracking, leading to financial discrepancies.

The [`deposit()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L39-L47) function uses unchecked to update the balance of the recipient:
```solidity
    function deposit(address to, bytes4 reason, string calldata comment) external payable {
        if (to == address(0)) revert InvalidAddressZero();

        unchecked {
>>          balanceOf[to] += msg.value;
        }

        emit Deposit(msg.sender, to, reason, msg.value, comment);
    }
```
This means that the addition operation does not automatically check for `overflow`, which can occur if `balanceOf[to]` is close to the maximum value for a `uint256`.

The function allows any user to deposit Ether to any address without restrictions. This means that anyone can increase the balance of any address. Without checks on the deposit amounts or the balance updates, there's no natural safeguard against reaching the `overflow threshold`.

## Impact
If an `overflow` occurs, the `balance` of the affected address could wrap around to a much smaller value or zero, effectively corrupting the balance tracking. This could lead to significant discrepancies in the contract's accounting.

## Recommendation
Remove the `unchecked` block and rely on Solidity's default `overflow checks` to prevent balance corruption.
```diff
    function deposit(address to, bytes4 reason, string calldata comment) external payable {
        if (to == address(0)) revert InvalidAddressZero();

-       unchecked {
-           balanceOf[to] += msg.value;
-       }

+       balanceOf[to] += msg.value;

        emit Deposit(msg.sender, to, reason, msg.value, comment);
    }
```

# [L-13] Premature `signature` expiration

## Vulnerability details
The current logic in [`_validateArtCreationSignature()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L592) is:
```solidity
if (expiresIn_ <= block.timestamp) revert SignatureExpired();
```
Given the error thrown, `SignatureExpired()`, it seems the intention is to allow the `signature` to be valid until the end of the specified `expiresIn_` timestamp. This means the `signature` should be valid throughout the block at which `expiresIn_` occurs, not just up to the moment before it.

This means that if `expiresIn_` is set to a specific time, the `signature` becomes invalid at the very start of that block, rather than at the end.

Other instances:
- [`createCred()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L254)
- [`updateCred()`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L295)

## Impact
Signatures may become invalid earlier than users anticipate, leading to failed transactions 

## Recommendation
To align with the likely intended behavior, the logic should be adjusted to:
```diff
-   if (expiresIn_ <= block.timestamp) revert SignatureExpired();
+   if (expiresIn_ < block.timestamp) revert SignatureExpired();
```
