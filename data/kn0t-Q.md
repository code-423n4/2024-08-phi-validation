# Quality Assurance for Phi Protocol
| Issue ID | Description |
| -------- | ----------- |
| [QA-01](#qa-01-missing-validations-in-updateartsettings-function) | Missing Validations in updateArtSettings Function |
| [QA-02](#qa-02-updateartsettings-function-not-protected-by-whennotpaused-modifier) | updateArtSettings Function Not Protected by whenNotPaused Modifier |
| [QA-03](#qa-03-potential-reentrancy-vulnerability-in-_handletrade-function) | Potential Reentrancy Vulnerability in _handleTrade Function |
| [QA-04](#qa-04-potential-reentrancy-vulnerability-in-distribute-function) | Potential Reentrancy Vulnerability in distribute Function |
| [QA-05](#qa-05-missing-length-validation-for-maxprices_-in-_validateandcalculatebatch) | Missing Length Validation for maxPrices_ in _validateAndCalculateBatch |
| [QA-06](#qa-06-insufficient-url-validation-in-updateartsettings) | Insufficient URL Validation in updateArtSettings |

## [QA-01] Missing Validations in updateArtSettings Function
### Impact
In the PhiFactory contract, users can call `createArt` with signed data from external sources, which can first validate the art settings before signing. However, the `updateArtSettings` function lacks certain validations that could lead to unintended behavior:

1. `startTime` and `endTime` can be set to values far in the future without any upper bound, potentially making art pieces unmintable for an extended period or effectively forever.
2. `startTime` and `endTime` can be set to the same value, making the art unmintable. This is inconsistent with the art creation validation in `_validateArtCreation`.
3. `endTime` can be set very close to `startTime`, making it difficult to mint the art within the given timeframe.

These issues could lead to user frustration and potentially render some art pieces unusable. The lack of these validations in `updateArtSettings` creates an inconsistency with the initial creation process, where external validation can be performed.

### Proof of Concept
The current implementation of `updateArtSettings` in PhiFactory:

[Link](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L215-L257)
```solidity
function updateArtSettings(
    uint256 artId_,
    string memory url_,
    address receiver_,
    uint256 maxSupply_,
    uint256 mintFee_,
    uint256 startTime_,
    uint256 endTime_,
    bool soulBounded_,
    IPhiNFT1155Ownable.RoyaltyConfiguration memory configuration
)
    external
    onlyArtCreator(artId_)
{
    if (receiver_ == address(0)) {
        revert InvalidAddressZero();
    }

    if (endTime_ < startTime_) {
        revert InvalidTimeRange();
    }
    if (endTime_ < block.timestamp) {
        revert EndTimeInPast();
    }

    // ... (rest of the function)
}
```

### Recommended Mitigation Steps
To maintain consistency with the initial art creation process and prevent potential issues, add additional validations to the `updateArtSettings` function:

1. Implement a reasonable upper bound for `startTime`.
2. Ensure a minimum time range between `startTime` and `endTime`.
3. Make the validation consistent with the art creation process.

Here's an example of how these validations could be implemented:

```solidity
uint256 constant END_THRESHOLD = 1 days;
uint256 constant MIN_MINT_RANGE = 3 days;
uint256 constant MAX_START_TIME = 365 days; // 1 year

function updateArtSettings(
    uint256 artId_,
    string memory url_,
    address receiver_,
    uint256 maxSupply_,
    uint256 mintFee_,
    uint256 startTime_,
    uint256 endTime_,
    bool soulBounded_,
    IPhiNFT1155Ownable.RoyaltyConfiguration memory configuration
)
    external
    onlyArtCreator(artId_)
{
    if (receiver_ == address(0)) {
        revert InvalidAddressZero();
    }

    if (startTime_ > block.timestamp + MAX_START_TIME) {
        revert StartTimeTooFarInFuture();
    }

    if (endTime_ <= startTime_ + MIN_MINT_RANGE) {
        revert InvalidTimeRange();
    }

    if (endTime_ < block.timestamp + END_THRESHOLD) {
        revert EndTimeInPast();
    }

    // ... (rest of the function)
}
```

These changes will ensure that:
1. The `startTime` cannot be set too far in the future.
2. There's a minimum time range between `startTime` and `endTime` for minting.
3. The `endTime` is always in the future with a reasonable threshold.

This will help prevent potential issues with unmintable or difficult-to-mint art pieces due to inappropriate time settings, and bring the update process more in line with the initial creation process where external validation can occur.


## [QA-02] updateArtSettings Function Not Protected by whenNotPaused Modifier
### Impact
The `updateArtSettings` function in the PhiFactory contract is not protected by the `whenNotPaused` modifier. This oversight allows art settings to be updated even when the contract is paused, potentially leading to inconsistencies in the contract state and bypassing intended restrictions during a paused state.

### Proof of Concept
The `updateArtSettings` function in PhiFactory lacks the `whenNotPaused` modifier:

[Link](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L215-L257)

```solidity
function updateArtSettings(
    uint256 artId_,
    string memory url_,
    address receiver_,
    uint256 maxSupply_,
    uint256 mintFee_,
    uint256 startTime_,
    uint256 endTime_,
    bool soulBounded_,
    IPhiNFT1155Ownable.RoyaltyConfiguration memory configuration
)
    external
    onlyArtCreator(artId_)
{
    // Function body
}
```

### Recommended Mitigation Steps
Add the `whenNotPaused` modifier to the `updateArtSettings` function to ensure it cannot be called when the contract is paused:





## [QA-03] Potential Reentrancy Vulnerability in _handleTrade Function
### Impact
In the Cred contract, the `_handleTrade` function, which is called by `buyShareCred` and `buyShareCredFor`, has a potential reentrancy vulnerability. The function returns excess ETH to the user before making a cross-contract call to PhiRewards to deposit the creator fee. While this doesn't currently pose an immediate threat, it could become a reentrancy point for cross-contract interactions in the future, potentially allowing manipulation of rewards.

### Proof of Concept
The vulnerable part of the `_handleTrade` function in the Cred contract:

[Link](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L588-L659)
```solidity
if (isBuy) {
    cred.currentSupply += amount_;
    uint256 excessPayment = msg.value - price - protocolFee - creatorFee;
    if (excessPayment > 0) {
        _msgSender().safeTransferETH(excessPayment); // Potential reentrancy point
    }
    lastTradeTimestamp[credId_][curator_] = block.timestamp;
} else {
    cred.currentSupply -= amount_;
    curator_.safeTransferETH(price - protocolFee - creatorFee);
}

protocolFeeDestination.safeTransferETH(protocolFee);
IPhiRewards(phiRewardsAddress).deposit{ value: creatorFee }( // Cross-contract call after potential reentrancy point
    creator, bytes4(keccak256("CREATOR_ROYALTY_FEE")), ""
);
```

### Recommended Mitigation Steps
To mitigate this potential vulnerability, consider one of the following approaches:

1. Move the excess ETH transfer to the end of the function, after all state changes and cross-contract calls:
2. Alternatively, add a nonReentrant modifier to the function to prevent any reentrancy:

Either of these approaches will help prevent potential future vulnerabilities related to reentrancy in cross-contract interactions.






## [QA-04] Potential Reentrancy Vulnerability in distribute Function
### Impact
In the CuratorRewardsDistributor contract, the `distribute` function is public and can be called by anyone. It sends an appreciation fee to the caller before making a cross-contract call to PhiRewards' `depositBatch` function. While this doesn't currently pose an immediate threat, it could become a reentrancy point for cross-contract interactions in the future, potentially allowing manipulation of rewards.

### Proof of Concept
The vulnerable part of the `distribute` function in the CuratorRewardsDistributor contract:

[Link](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L77-L130)

```solidity
balanceOf[credId] -= totalBalance;

_msgSender().safeTransferETH(royaltyfee + distributeAmount - actualDistributeAmount);

//slither-disable-next-line arbitrary-send-eth
phiRewardsContract.depositBatch{ value: actualDistributeAmount }(
    distributeAddresses, amounts, reasons, "deposit from curator rewards distributor"
);
```

### Recommended Mitigation Steps
To mitigate this potential vulnerability, consider one of the following approaches:
1. Move the appreciation fee transfer to the end of the function, after all state changes and cross-contract calls:
2. Alternatively, add a nonReentrant modifier to the function to prevent any reentrancy:


Either of these approaches will help prevent potential future vulnerabilities related to reentrancy in cross-contract interactions and protect against possible reward manipulation.





## [QA-05] Missing Length Validation for maxPrices_ in _validateAndCalculateBatch
### Impact
In the Cred contract, the `batchSellShareCred` and `batchBuyShareCred` functions call the `_validateAndCalculateBatch` function with `credIds_`, `amounts_`, and `maxPrices_` as parameters. While the lengths of `credIds_` and `amounts_` are validated to be the same, there is no validation for the length of `maxPrices_`. This oversight could lead to unexpected behavior or errors if the `maxPrices_` array has a different length than the other input arrays.

### Proof of Concept
The vulnerable part of the `_validateAndCalculateBatch` function in the Cred contract:

[Link](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L810-L831)

```solidity
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
    if (length != amounts_.length) {
        revert InvalidArrayLength();
    }
    if (length == 0) {
        revert EmptyBatchOperation();
    }
    // Missing validation for priceLimits_ length
    ...
}
```

### Recommended Mitigation Steps
Add a validation check for the length of `priceLimits_` to ensure it matches the length of `credIds_` and `amounts_`:

```solidity
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
    if (length != amounts_.length || length != priceLimits_.length) {
        revert InvalidArrayLength();
    }
    if (length == 0) {
        revert EmptyBatchOperation();
    }
    ...
}
```

## [QA-06] Insufficient URL Validation in updateArtSettings
### Impact
In the PhiFactory contract, the `updateArtSettings` function allows art creators to specify an arbitrary URL for their art without proper validation. This lack of validation could potentially lead to several issues:

1. Storage bloat and increased gas costs due to excessively long URLs.
2. Potential bugs in the UI when rendering or processing invalid URLs.
3. Possible security vulnerabilities such as XSS (Cross-Site Scripting) or other injection methods if the URL is used directly in a front-end application without proper sanitization.

Unlike the `createArt` function, which uses signed data from external sources that can validate the art settings before signing, `updateArtSettings` lacks these checks, creating an inconsistency in the validation process.

### Proof of Concept
The `updateArtSettings` function in PhiFactory lacks URL validation:

[Link](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L215-L257)

```solidity
function updateArtSettings(
    uint256 artId_,
    string memory url_,
    address receiver_,
    uint256 maxSupply_,
    uint256 mintFee_,
    uint256 startTime_,
    uint256 endTime_,
    bool soulBounded_,
    IPhiNFT1155Ownable.RoyaltyConfiguration memory configuration
)
    external
    onlyArtCreator(artId_)
{
    // ... other validations ...

    art.uri = url_; // No validation on url_

    // ... rest of the function ...
}
```

### Recommended Mitigation Steps
Implement proper URL validation in the `updateArtSettings` function:

1. Add a maximum length check for the URL to prevent storage bloat.
2. Implement basic protocol validation (e.g., ensure it starts with "http://" or "https://").
3. Consider using a whitelist of allowed domains or IPFS/Arweave gateways.

Example implementation:

```solidity
function updateArtSettings(
    uint256 artId_,
    string memory url_,
    // ... other parameters ...
)
    external
    onlyArtCreator(artId_)
{
    // ... other validations ...

    require(bytes(url_).length <= MAX_URL_LENGTH, "URL too long");
    require(
        LibString.startsWith(url_, "https://") || 
        LibString.startsWith(url_, "http://") ||
        LibString.startsWith(url_, "ipfs://"),
        "Invalid URL protocol"
    );

    // Optional: Check against a whitelist of allowed domains
    require(isAllowedDomain(url_), "Domain not allowed");

    art.uri = url_;

    // ... rest of the function ...
}
```