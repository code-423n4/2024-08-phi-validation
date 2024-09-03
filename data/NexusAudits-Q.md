# QA Report: Phi Protocol

## L1: Potential DoS Vulnerability in `depositBatch` Function

### Lines of Code:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L54

### Summary
The `depositBatch` function in the `RewardControl` contract iterates over the `recipients`, `amounts`, and `reasons` arrays without any restrictions on their lengths. This could lead to excessive gas consumption and potential Denial of Service (DoS) attacks if an attacker submits a transaction with overly large arrays.

### Impact 

An attacker could create a transaction that consumes excessive gas, potentially leading to a DoS situation where legitimate users are unable to execute their transactions.

### Affected Code Snippet
```solidity
function depositBatch(
    address[] calldata recipients,
    uint256[] calldata amounts,
    bytes4[] calldata reasons,
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
    for (uint256 i = 0; i < numRecipients; i++) {
        expectedTotalValue += amounts[i];
    }

    if (msg.value != expectedTotalValue) {
        revert InvalidDeposit();
    }

    for (uint256 i = 0; i < numRecipients; i++) {
        address recipient = recipients[i];
        uint256 amount = amounts[i];

        if (recipient == address(0)) {
            revert InvalidAddressZero();
        }

        balanceOf[recipient] += amount;

        emit Deposit(msg.sender, recipient, reasons[i], amount, comment);
    }
}
```

### Suggested Fix
Consider adding input validation to restrict the maximum allowable length for the `recipients`, `amounts`, and `reasons` arrays. For example:

### Proposed Code 
```solidity
uint256 constant MAX_BATCH_SIZE = 100; // Set a reasonable limit

function depositBatch(
    address[] calldata recipients,
    uint256[] calldata amounts,
    bytes4[] calldata reasons,
    string calldata comment
)
    external
    payable
{
    uint256 numRecipients = recipients.length;

    if (numRecipients > MAX_BATCH_SIZE) {
        revert ArrayLengthExceeded();
    }

    if (numRecipients != amounts.length || numRecipients != reasons.length) {
        revert ArrayLengthMismatch();
    }

    uint256 expectedTotalValue;
    for (uint256 i = 0; i < numRecipients; i++) {
        expectedTotalValue += amounts[i];
    }

    if (msg.value != expectedTotalValue) {
        revert InvalidDeposit();
    }

    for (uint256 i = 0; i < numRecipients; i++) {
        address recipient = recipients[i];
        uint256 amount = amounts[i];

        if (recipient == address(0)) {
            revert InvalidAddressZero();
        }

        balanceOf[recipient] += amount;

        emit Deposit(msg.sender, recipient, reasons[i], amount, comment);
    }
}
```

## L2: Potential DoS Vulnerability in Multiple Functions in Cred Contract

### Lines of Code:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L897

### Summary
The Cred.sol contract contains several functions that iterate over data structures, specifically in the `_getCuratorData`, `getPositionsForCurator`, and `batch` functions. Without proper input validation, the sizes of these data structures (e.g., `shareBalance`, `_credIdsPerAddress`) could become excessively large, leading to high gas consumption and potential Denial of Service (DoS) attacks.

### Impact

An attacker could exploit this by submitting transactions with large data structures, causing excessive gas consumption and potentially halting legitimate transactions.

### Affected Code Snippets
1. **_getCuratorData Function**
   ```solidity
   function getCuratorAddresses(
       uint256 credId_,
       uint256 start_,
       uint256 stop_
   ) public view returns (address[] memory) {
       CuratorData[] memory curatorData = _getCuratorData(credId_, start_, stop_);
       address[] memory result = new address[](curatorData.length);
       for (uint256 i = 0; i < curatorData.length; i++) {
           result[i] = curatorData[i].curator;
       }
       return result;
   }
   ```

2. **getPositionsForCurator Function**
   ```solidity
   function getPositionsForCurator(
       address curator_,
       uint256 start_,
       uint256 stop_
   ) external view returns (uint256[] memory credIds, uint256[] memory amounts) {
       uint256[] storage userCredIds = _credIdsPerAddress[curator_];
       uint256 stopIndex;
       if (userCredIds.length == 0) {
           return (credIds, amounts);
       }
       if (stop_ == 0 || stop_ > userCredIds.length) {
           stopIndex = userCredIds.length;
       } else {
           stopIndex = stop_;
           if (start_ >= stopIndex) {
               revert InvalidPaginationParameters();
           }
       }
       credIds = new uint256[](stopIndex - start_);
       amounts = new uint256[](stopIndex - start_);
       uint256 index = 0;
       for (uint256 i = start_; i < stopIndex; i++) {
           uint256 credId = userCredIds[i];
           if (_credIdExistsPerAddress[curator_][credId]) {
               uint256 amount = shareBalance[credId].get(curator_);
               credIds[i] = credId;
               amounts[i] = amount;
               index++;
           }
       }
       // Resize the result array to remove unused slots
       assembly {
           mstore(credIds, index)
           mstore(amounts, index)
       }
   }
   ```

## Suggested Fix
Consider adding input validation to restrict the maximum allowable size for the data structures being iterated. For example:

### Proposed Code Modification
```solidity
uint256 constant MAX_CRED_IDS_LENGTH = 100; // Set a reasonable limit

function getPositionsForCurator(
    address curator_,
    uint256 start_,
    uint256 stop_
) external view returns (uint256[] memory credIds, uint256[] memory amounts) {
    uint256[] storage userCredIds = _credIdsPerAddress[curator_];

    if (userCredIds.length > MAX_CRED_IDS_LENGTH) {
        revert ArrayLengthExceeded();
    }

    // Existing logic follows...
}
```

## Conclusion
Implementing input validation for the sizes of the iterated data structures in the affected functions will enhance the contract's security and prevent potential DoS attacks.

## L3: Flawed Reentrancy Mechanism in Cred.sol

### Lines of Code:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L112

### Summary
The `nonReentrant` modifier aims to prevent such attacks by using a `locked` variable:

```solidity
modifier nonReentrant() virtual {
    if (locked != 1) revert Reentrancy();
    locked = 2;
    _; // Execute the function body
    locked = 1;
}
```

The intended logic is as follows:

1. **Check if `locked` is 1:** If `locked` is not 1, it means a reentrant call is attempting to execute, and the transaction is reverted.
2. **Set `locked` to 2:** If `locked` is 1, the function proceeds, and `locked` is set to 2 to indicate that the function is currently executing.
3. **Execute the function body:** The core logic of the function is executed.
4. **Reset `locked` to 1:** After the function body completes, `locked` is reset to 1, allowing subsequent calls to enter.


This implementation is flawed because of the timing of the `locked` variable updates.  The `locked` variable is set to 2 within the `nonReentrant` modifier, *before* the function body is executed.  This creates a window of vulnerability:

If, during the execution of the function body (step 3 above), an external contract makes a reentrant call back into the same function or another function protected by `nonReentrant`, the `if (locked != 1)` check will still pass. This is because `locked` was set to 2 earlier in the modifier, and it hasn't been reset to 1 yet.

This allows the reentrant call to proceed, potentially leading to unintended state changes or unauthorized actions.

### Impact

The current implementation fails to effectively block reentrant calls that occur during the execution of the protected function.

### Recommendation

Use a well-established reentrancy guard library like OpenZeppelin's `ReentrancyGuardUpgradeable`. These libraries use more robust mechanisms, such as mutex locks, to ensure that reentrant calls are correctly blocked throughout the entire execution of the protected function.

## L4: Malicious contract upgrades is possible

### Lines of Code:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L937

### Summary
The `_authorizeUpgrade` function, inherited from the `UUPSUpgradeable` contract, allows the contract owner to upgrade the contract to a new implementation without any additional safeguards. This poses a significant risk, as a compromised owner account or a malicious upgrade could lead to severe consequences, including:

* **Loss of Funds:**  The new implementation could introduce malicious code that drains the contract's funds or manipulates user balances.
* **Unauthorized Access:** The new implementation could grant unauthorized access to sensitive functions or data, compromising the integrity of the system
* **Denial of Service:** The new implementation could contain bugs or errors that render the contract unusable, disrupting its intended functionality

### Vulnerable Code

```solidity
function _authorizeUpgrade(address newImplementation) internal override onlyOwner {
    // This function is intentionally left empty to allow for upgrades
}
```

### Impact

This vulnerability could have a catastrophic impact on the contract and its users, potentially leading to the loss of funds, unauthorized access, or complete denial of service.

### Recommendation

- Introduce a time delay between the initiation of an upgrade and its execution. This provides a window for users to withdraw their funds or take other protective measures if they suspect a malicious upgrade
- Require multiple signatures from a group of trusted parties to authorize an upgrade. This makes it more difficult for an attacker to compromise the upgrade process
- Implement a mechanism to pause or halt the contract's functionality in case of a suspected malicious upgrade

## L5: Missing `nonReentrant` Modifier in `claimFromFactory` Function

### Lines of Code:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L164

### Summary
The `PhiNFT1155` contract manages the creation and claiming of art tokens from the Phi Factory contract. This contract is built on top of ERC1155 and incorporates additional features such as pausing, reentrancy protection, and royalty management. 

The `claimFromFactory` function allows a user to claim an art token by minting it and handling associated rewards. However, the function lacks the `nonReentrant` modifier, exposing it to potential reentrancy attacks.

The `claimFromFactory` function transfers Ether to various external addresses before completing its execution. Because it lacks the `nonReentrant` modifier, this function can be called again before the first invocation completes, allowing a potential reentrancy attack.

### Code Snippet
```solidity
function claimFromFactory(
    uint256 artId_,
    address minter_,
    address ref_,
    address verifier_,
    uint256 quantity_,
    bytes32 data_,
    string calldata imageURI_
)
    external
    payable
    whenNotPaused
    onlyPhiFactory
{
    uint256 tokenId_ = _artIdToTokenId[artId_];
    if (tokenId_ == 0) {
        revert InValdidTokenId();
    }
    mint(minter_, tokenId_, quantity_, imageURI_, data_);
    address aristRewardReceiver = phiFactoryContract.artData(artId_).receiver;
    bytes memory addressesData_ = abi.encode(minter_, aristRewardReceiver, ref_, verifier_);

    IPhiRewards(payable(phiFactoryContract.phiRewardsAddress())).handleRewardsAndGetValueSent{ value: msg.value }(
        artId_, credId, quantity_, mintFee(tokenId_), addressesData_, credChainId == block.chainid
    );
    emit ArtClaimedData(minter_, aristRewardReceiver, ref_, verifier_, artId_, tokenId_, quantity_, data_);
}
```

### Impact

- An attacker could exploit this vulnerability by initiating a reentrancy attack, allowing them to claim multiple tokens or drain the contract's funds.
- This could lead to significant financial loss and undermine the integrity of the art creation and claiming process.


### Scenario
1. **User calls `claimFromFactory`:** The function starts executing and begins by minting a token and transferring Ether to the reward contract.
2. **External Call:** During the Ether transfer, a malicious contract could re-enter `claimFromFactory` before the first execution completes, causing the function to execute again with the same or different parameters.
3. **Result:** The attacker could drain the contract or claim more tokens than intended.


### Recommendation:

Add the nonReentrant modifier to the claimFromFactory function.

## L6: Unbounded `for` Loop in `safeBatchTransferFrom`

### Lines of Code:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L332

### Summary

The `PhiNFT1155` contract extends the ERC1155 standard to handle various token transfer operations. The `safeBatchTransferFrom` function allows transferring multiple tokens in a single transaction by iterating over the `ids_` array.

The `safeBatchTransferFrom` function contains a `for` loop that iterates over the entire `ids_` array:

```solidity
  for (uint256 i; i < ids_.length; i++) {
      if (from_ != address(0) && soulBounded(ids_[i])) revert TokenNotTransferable();
  }
```

If the `ids_` array is very large, the loop could consume a significant amount of gas, potentially causing the transaction to fail due to exceeding the block gas limit.

### Impact

Transactions with large `ids_` arrays could be excessively expensive or fail, limiting the contract's usability for bulk transfers. This could also expose users to unnecessary gas costs for operations that might not complete.

### Scenario

A user attempts to transfer a large batch of tokens using `safeBatchTransferFrom`. If the `ids_` array is extensive, the transaction could fail due to hitting the gas limit, or it might consume an unreasonable amount of gas, making it impractical for users.

### Recommendation

- Implement a mechanism to limit the number of tokens that can be transferred in a single batch. This could involve setting a maximum allowed array length for `ids_`.
- Alternatively, consider a pagination-like approach, where large transfers are split across multiple transactions, allowing users to handle large batches more efficiently without exceeding gas limits.

## L7: Typo in `InValdidTokenId` Error

### Lines of Code:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L180

### Summary
The `PhiNFT1155` contract manages various aspects of creating and claiming art tokens within the Phi ecosystem. During these operations, specific error messages are used to handle exceptional cases.

The contract includes a custom error named `InValdidTokenId`. This error is triggered when an invalid token ID is referenced during operations such as claiming or minting. However, the error name contains a typo—"InValdid" instead of "Invalid."
  
```solidity
revert InValdidTokenId();
```

### Impact 

This typo may cause confusion during debugging or when reviewing the contract code. It also affects code readability and consistency, which could lead to misunderstandings or mistakes when interacting with the contract or referencing this error in other systems or documentation.

### Scenario

If the function `claimFromFactory` is called with a non-existent `artId`, the contract will revert with the typo `InValdidTokenId`. Developers or users encountering this error might be momentarily confused due to the typo.

### Fix

Rename the error to `InvalidTokenId` to accurately reflect its purpose and maintain code clarity.

```solidity
revert InvalidTokenId();
```

This change will reduce potential confusion during contract interactions.


## L8: Unchecked Increment of `tokenIdCounter` in `createArtFromFactory`

### Lines of Code:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L148

### Summary
The `PhiNFT1155` contract allows the creation of art tokens through the `createArtFromFactory` function. This function utilizes the `tokenIdCounter` to assign unique token IDs to new art pieces.

Within the `createArtFromFactory` function, the `tokenIdCounter` is incremented inside an `unchecked` block:

  ```solidity
  unchecked {
      tokenIdCounter += 1;
  }
  ```

While the likelihood of `tokenIdCounter` overflowing is low, especially in the near future, it's a good practice to include overflow checks. This is particularly important when dealing with ID counters that could be used in calculations or operations elsewhere in the contract. An unchecked overflow could lead to unpredictable behavior or security vulnerabilities.

### Impact

If an overflow were to occur, it could result in duplicated token IDs or other unintended consequences, potentially disrupting the functionality of the contract and affecting the integrity of token ownership records.

### Scenario
Over time, if the `tokenIdCounter` were to approach its maximum value and an overflow occurs, the counter would wrap around to zero, leading to conflicts with existing token IDs. This could cause severe issues in token management and transfer operations.

### Recommendation

- Implement an overflow check before incrementing `tokenIdCounter`. The simplest approach is to use Solidity's built-in overflow checks, which are enabled by default in versions 0.8.0 and later.
- You can also explicitly revert the transaction if the increment would cause an overflow, ensuring that the contract handles the situation gracefully:

```solidity
require(tokenIdCounter + 1 > tokenIdCounter, "Overflow error: tokenIdCounter exceeded its limit");
tokenIdCounter += 1;
```

This check would ensure that any attempt to exceed the maximum value of `tokenIdCounter` is caught early, maintaining the contract's robustness.


## L9: Remove the unused console2 import

### Lines of Code:
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/curve/BondingCurve.sol#L9

### Summary
The `BondingCurve` contract imports `console2` from the `forge-std` library. 

The imported `console2` is not used anywhere in the contract. This results in unnecessary code, potentially increasing deployment size and reducing code clarity.

### Impact
- Increased deployment size, albeit minimal.
- Reduced code readability and maintainability.

### Recommendation
Remove the unused `console2` import statement to clean up the code and optimize the contract.

## L10: Typo in `initilaized` Variable Name

### Lines of Code:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/CreatorRoyaltiesControl.sol#L13

### Summary
The `CreatorRoyaltiesControl` contract defines a boolean variable `initilaized` to track whether the contract has been initialized. The variable name `initilaized` contains a typo and should be spelled `initialized`. This can lead to confusion and reduces code readability.

## Impact
- Reduces code clarity and readability.
- Could potentially cause issues in maintenance or further development.

## Recommendation
- Rename the `initilaized` variable to `initialized` throughout the contract.
- Ensure that all references to this variable are updated to reflect the corrected spelling.