# QA for Phi


## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-01](#qa-01-sellers-could-be-underpaid-due-to-pricelimit-condition-set-in-_handletrade-function) | Sellers could be underpaid due to `priceLimit` condition set in `_handleTrade` function |
| [QA-02](#qa-02-referral-reward-incorrectly-added-to-artists-balance-when-referral-address-is-zero) | Referral reward incorrectly added to artist's balance when referral address is zero |
| [QA-03](#qa-03-funds-could-be-lost-due-to-incorrect-handling-of-rounding-errors-in-reward-distribution) | Funds could be lost due to incorrect handling of rounding errors in reward distribution |
| [QA-04](#qa-04-supportsinterface-function-does-not-correctly-implement-erc165) | `supportsInterface` function does not correctly implement ERC165 |
| [QA-05](#qa-05-potential-precision-loss-in-bondingcurve_curve) | Potential precision loss in `BondingCurve::_curve` |
| [QA-06](#qa-06-flawed-data-structure-in-_removecredidperaddress-function-when-removing-last-element) | Flawed data structure in `_removeCredIdPerAddress` function when removing last element |
| [QA-07](#qa-07-missing-whennotpaused-modifier-in-safetransferfrom-and-safebatchtransferfrom-functions-allows-token-transfers-when-contract-is-paused) | Missing `whenNotPaused` modifier in `safeTransferFrom` and `safeBatchTransferFrom` functions allows token transfers when contract is paused |
| [QA-08](#qa-08-misspelled-initialized-variable-compromises-initialization-check) | Misspelled 'initialized' variable compromises initialization check |

## [QA-01] Sellers could be underpaid due to `priceLimit` condition set in `_handleTrade` function

### Impact
The implementation of the `_handleTrade` function may finalize in sellers receiving less than their specified `priceLimit` when selling shares. This is because the logic does not sufficiently account for the deduction of protocol and creator fees from the sale price leading to underpayment to the seller.

### Recommended Mitigation Steps
To ensure that the seller always receives at least the `priceLimit` after fees are deducted, the logic should be adjusted like this:

```diff
if (isBuy) {
    if (priceLimit != 0 && price + protocolFee + creatorFee > priceLimit) revert PriceExceedsLimit();
    if (supply + amount_ > MAX_SUPPLY) {
        revert MaxSupplyReached();
    }

    if (msg.value < price + protocolFee + creatorFee) {
        revert InsufficientPayment();
    }
} else {
-    if (priceLimit != 0 && price - protocolFee - creatorFee < priceLimit) revert PriceBelowLimit();
+    if (priceLimit != 0 && price < priceLimit + protocolFee + creatorFee) revert PriceBelowLimit();
    if (block.timestamp <= lastTradeTimestamp[credId_][curator_] + SHARE_LOCK_PERIOD) {
        revert ShareLockPeriodNotPassed(
            block.timestamp, lastTradeTimestamp[credId_][curator_] + SHARE_LOCK_PERIOD
        );
    }
    (, uint256 nums) = shareBalance[credId_].tryGet(curator_);
    if (nums < amount_) {
        revert InsufficientShares();
    }
}
```



## [QA-02] Referral reward incorrectly added to artist's balance when referral address is zero

### Impact
When the referral address is the zero address, the referral reward is incorrectly added to the artist's balance. Hence the artist ends up receiving additional rewards that should not be paid out.

### Proof of Concept
Take a look at the [`depositRewards`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L78-L121) function: 
```solidity
if (referral_ == minter_ || referral_ == address(0)) {
    artistTotalReward_ += referralTotalReward_;
    referralTotalReward_ = 0;
} else if (referral_ != address(0)) {
    balanceOf[referral_] += referralTotalReward_;
}
```

The nub of matter is that when `referral_` is the zero address, the `referralTotalReward_` is added to `artistTotalReward_`, which is then added to the artist's balance (`balanceOf[receiver_]`). 

However, if the referral address is zero, it likely means there is no referral, and the referral reward should not be paid out at all. Instead, the contract is effectively giving the referral reward to the artist when the referral address is zero.


### Recommended Mitigation Steps
Consider modifying the logic to only add the referral reward to the referral's balance if the referral address is non-zero and different from the minter's address. If there is no valid referral address, the referral reward should not be paid out to anyone.

```diff
function depositRewards(
    uint256 artId_,
    uint256 credId_,
    bytes calldata addressesData_,
    uint256 artistTotalReward_,
    uint256 referralTotalReward_,
    uint256 verifierTotalReward_,
    uint256 curateTotalReward_,
    bool chainSync_
)
    internal
{
    (address minter_, address receiver_, address referral_, address verifier_) =
        abi.decode(addressesData_, (address, address, address, address));

    if (receiver_ == address(0) || minter_ == address(0) || verifier_ == address(0)) {
        revert InvalidAddressZero();
    }

-    if (referral_ == minter_ || referral_ == address(0)) {
-        artistTotalReward_ += referralTotalReward_;
-        referralTotalReward_ = 0;
-    } else if (referral_ != address(0)) {
+    if (referral_ != address(0) && referral_ != minter_) {
        balanceOf[referral_] += referralTotalReward_;
+    } else {
+        referralTotalReward_ = 0;
    }

    balanceOf[verifier_] += verifierTotalReward_;
    balanceOf[receiver_] += artistTotalReward_;

    bytes memory rewardsData;
    if (chainSync_ && address(curatorRewardsDistributor) != address(0)) {
        //slither-disable-next-line arbitrary-send-eth
        curatorRewardsDistributor.deposit{ value: curateTotalReward_ }(credId_, curateTotalReward_);
        rewardsData = abi.encode(artistTotalReward_, referralTotalReward_, verifierTotalReward_, curateTotalReward_);
    } else {
        balanceOf[receiver_] += curateTotalReward_;
        rewardsData =
            abi.encode(artistTotalReward_ + curateTotalReward_, referralTotalReward_, verifierTotalReward_, 0);
    }

    bytes memory credData = abi.encode(artId_, credId_, chainSync_);

    emit RewardsDeposit(credData, minter_, receiver_, referral_, verifier_, rewardsData);
}
```









## [QA-03] Funds could be lost due to incorrect handling of rounding errors in reward distribution

### Impact
This could result in unintended recipients receiving excess ETH due to rounding errors during reward distribution.

### Proof of Concept
Take a look at the [`distribute`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L105-L121) function: 

```solidity
function distribute(uint256 credId) external {
    // ... (previous code)

    uint256 actualDistributeAmount = 0;
    for (uint256 i = 0; i < distributeAddresses.length; i++) {
        address user = distributeAddresses[i];

        uint256 userAmounts = credContract.getShareNumber(credId, user);
        uint256 userRewards = (distributeAmount * userAmounts) / totalNum;

        if (userRewards > 0) {
            amounts[i] = userRewards;
            actualDistributeAmount += userRewards;
        }
    }

    balanceOf[credId] -= totalBalance;

    _msgSender().safeTransferETH(royaltyfee + distributeAmount - actualDistributeAmount);  <-------------

    // ... (rest of the function)
}
```

The meat of the matter lies in the last line before the comment block. The contract calculates the difference between the total amount available for distribution (`royaltyfee + distributeAmount`) and the actual amount distributed (`actualDistributeAmount`). This difference, which includes any rounding errors, is then sent to `_msgSender()`.

This approach is flawed because `_msgSender()` is the address calling the `distribute` function, which may not be the intended recipient of these excess funds. In many cases, this could be a regular user rather than the contract owner or administrator.

Furthermore, this design allows for potential exploitation. A malicious actor could repeatedly call the `distribute` function, accumulating small amounts of excess ETH each time, especially if they have control over the timing of when the function is called.

### Recommended Mitigation Steps
Consider modifying the contract to handle the remaining balance appropriately. Instead of sending it to `_msgSender()`, consider one of these approaches: keep the remaining balance in the contract for future distributions, or send it to the contract owner or a designated treasury address.












## [QA-04] `supportsInterface` function does not correctly implement ERC165

### Impact
The implementation of the `supportsInterface` function only checks for the `IERC2981` interface and does not account for the `IERC165` interface or any other interfaces that the contract might implement. This can lead to incorrect interface detection, causing the contract to be incompatible with systems that rely on ERC165 for interface detection. This could result in the contract not being recognized as supporting the interfaces it actually implements.


### Proof of Concept
The `supportsInterface` function is as follows:
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/CreatorRoyaltiesControl.sol#L71-L73


```solidity
function supportsInterface(bytes4 interfaceId) public view virtual returns (bool) {
    return interfaceId == type(IERC2981).interfaceId;
}
```

This implementation only checks for the `IERC2981` interface and does not account for the `IERC165` interface, which is required for proper ERC165 support. Additionally, it does not allow for the contract to support multiple interfaces, which could be necessary if the contract is intended to implement other interfaces like ERC1155.

### Recommended Mitigation Steps
The `supportsInterface` function should be updated to check for both the `IERC2981` and `IERC165` interfaces. Additionally, it should allow for the possibility of supporting other interfaces by using the `super` keyword.

```diff
function supportsInterface(bytes4 interfaceId) public view virtual returns (bool) {
-    return interfaceId == type(IERC2981).interfaceId;
+    return
+        interfaceId == type(IERC2981).interfaceId ||
+        interfaceId == type(IERC165).interfaceId ||
+        super.supportsInterface(interfaceId);
}
```














## [QA-05] Potential precision loss in `BondingCurve::_curve`

### Impact
The implementation of the `_curve` function in the BondingCurve contract may lead to precision loss and potential rounding errors, especially for small values of `targetAmount_`. This could result in inaccurate price calculations.

### Proof of Concept
Here's the `_curve` function: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/curve/BondingCurve.sol#L117-L120

```solidity
function _curve(uint256 targetAmount_) private pure returns (uint256) {
    return (TOTAL_SUPPLY_FACTOR * CURVE_FACTOR * 1 ether) / (TOTAL_SUPPLY_FACTOR - targetAmount_)
        - CURVE_FACTOR * 1 ether - INITIAL_PRICE_FACTOR * targetAmount_ / 1000;
}
```

The last part of the calculation, `INITIAL_PRICE_FACTOR * targetAmount_ / 1000`, performs division before multiplication. For small values of `targetAmount_`, this could result in the division rounding down to zero, effectively negating the impact of `INITIAL_PRICE_FACTOR` on the price calculation.

For example, if `targetAmount_` is less than 1000, the result of `targetAmount_ / 1000` will be 0, regardless of the value of `INITIAL_PRICE_FACTOR`. This means that for small purchases or sales, the initial price factor won't be correctly accounted for in the curve calculation.

### Recommended Mitigation Steps
The order of operations in the last part of the `_curve` function should be changed to perform multiplication before division.

```diff
function _curve(uint256 targetAmount_) private pure returns (uint256) {
    return (TOTAL_SUPPLY_FACTOR * CURVE_FACTOR * 1 ether) / (TOTAL_SUPPLY_FACTOR - targetAmount_)
-        - CURVE_FACTOR * 1 ether - INITIAL_PRICE_FACTOR * targetAmount_ / 1000;
+        - CURVE_FACTOR * 1 ether - (INITIAL_PRICE_FACTOR * targetAmount_) / 1000;
}
```














## [QA-06] Flawed data structure in `_removeCredIdPerAddress` function when removing last element

### Impact
This issue can cause inconsistency between the `_credIdsPerAddress` array and the `_credIdsPerAddressCredIdIndex` mapping. When removing the last element from a user's credId list, the function unnecessarily updates the index and then deletes it, potentially causing errors in future operations that rely on this index mapping. 

### Proof of Concept
Here's the `_removeCredIdPerAddress` function: 
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L695-L729

```solidity
function _removeCredIdPerAddress(uint256 credId_, address sender_) public {
    // ... (earlier code omitted for brevity)

    uint256 lastIndex = _credIdsPerAddress[sender_].length - 1;
    uint256 lastCredId = _credIdsPerAddress[sender_][lastIndex];
    _credIdsPerAddress[sender_][indexToRemove] = lastCredId;

    if (indexToRemove < lastIndex) {
        _credIdsPerAddressCredIdIndex[sender_][lastCredId] = indexToRemove;
    }

    _credIdsPerAddress[sender_].pop();

    delete _credIdsPerAddressCredIdIndex[sender_][credIdToRemove];

    // rest of the code omitted for brevity
}
```

1. When `credId_` is the last element in the array:
   - `indexToRemove` equals `lastIndex`
   - `lastCredId` equals `credIdToRemove`
2. The function unnecessarily assigns `lastCredId` to its own position in the array.
3. The `if` condition is false, so the index update is skipped.
4. The last element is removed from the array.
5. The index mapping for the removed credId is deleted.

#### Scenario
Let's say a user has credIds [1, 2, 3], and we want to remove credId 3:
1. The function finds 3 at index 2.
2. It assigns 3 to index 2 (unnecessary operation).
3. It removes 3 from the array, resulting in [1, 2].
4. It deletes the index mapping for 3.

The array is correct, but the index mapping for 3 is deleted, which could cause issues if any other part of the protocol tries to look up the index for credId 3 for this user.

### Recommended Mitigation Steps
Modify the `_removeCredIdPerAddress` function to handle the last element case separately:

```diff
function _removeCredIdPerAddress(uint256 credId_, address sender_) public {
    // ... (earlier code remains unchanged)

    uint256 lastIndex = _credIdsPerAddress[sender_].length - 1;

-   uint256 lastCredId = _credIdsPerAddress[sender_][lastIndex];
-   _credIdsPerAddress[sender_][indexToRemove] = lastCredId;
-
-   if (indexToRemove < lastIndex) {
+   if (indexToRemove != lastIndex) {
+       uint256 lastCredId = _credIdsPerAddress[sender_][lastIndex];
+       _credIdsPerAddress[sender_][indexToRemove] = lastCredId;
        _credIdsPerAddressCredIdIndex[sender_][lastCredId] = indexToRemove;
    }

    _credIdsPerAddress[sender_].pop();

    delete _credIdsPerAddressCredIdIndex[sender_][credIdToRemove];

    // rest of the code remains unchanged
}
```














## [QA-07] Missing `whenNotPaused` modifier in `safeTransferFrom` and `safeBatchTransferFrom` functions allows token transfers when contract is paused

### Impact
The absence of the `whenNotPaused` modifier in the `safeTransferFrom` and `safeBatchTransferFrom` functions allows users to transfer tokens even when the contract is paused. This undermines the purpose of the pause functionality, which is intended to halt critical operations.

### Proof of Concept

#### `safeTransferFrom` Function
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L307-L324
The `safeTransferFrom` function is responsible for transferring tokens from one address to another. However, it lacks the `whenNotPaused` modifier, which means it can still be executed even when the contract is paused:

```solidity
function safeTransferFrom(
    address from_,
    address to_,
    uint256 id_,
    uint256 value_,
    bytes memory data_
) public override {
    if (from_ != address(0) && soulBounded(id_)) revert TokenNotTransferable();
    address sender = _msgSender();
    if (from_ != sender && !isApprovedForAll(from_, sender)) {
        revert ERC1155MissingApprovalForAll(sender, from_);
    }

    _safeTransferFrom(from_, to_, id_, value_, data_);
}
```

#### `safeBatchTransferFrom` Function
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L332-L351
Similarly, the `safeBatchTransferFrom` function, which handles batch transfers of tokens, also lacks the `whenNotPaused` modifier:

```solidity
function safeBatchTransferFrom(
    address from_,
    address to_,
    uint256[] memory ids_,
    uint256[] memory values_,
    bytes memory data_
) public override {
    for (uint256 i; i < ids_.length; i++) {
        if (from_ != address(0) && soulBounded(ids_[i])) revert TokenNotTransferable();
    }
    address sender = _msgSender();
    if (from_ != sender && !isApprovedForAll(from_, sender)) {
        revert ERC1155MissingApprovalForAll(sender, from_);
    }
    _safeBatchTransferFrom(from_, to_, ids_, values_, data_);
}
```

The `whenNotPaused` modifier is intended to prevent certain functions from being executed when the contract is paused. Without this modifier, users can still transfer tokens even when the contract is paused, which could lead to unintended consequences during periods when the contract is supposed to be inactive.

### Recommended Mitigation Steps
Add the `whenNotPaused` modifier to both the `safeTransferFrom` and `safeBatchTransferFrom` functions to ensure that token transfers are not allowed when the contract is paused. 















## [QA-08] Misspelled 'initialized' variable compromises initialization check

### Proof of Concept
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/CreatorRoyaltiesControl.sol
Grep the contract for the spelling `initilaized` and all the instances will come up.

### Recommended Mitigation Steps
Correct the spelling of 'initilaized' to 'initialized' throughout the contract. 







