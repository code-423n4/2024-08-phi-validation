


###  Invalid Array Length Handling in Batch Buy Function (Root Cause + Impact)

**Description:**

The function `batchBuyShareCred` is expected to handle batch operations by accepting three arrays: `credIds_`, `amounts_`, and `maxPrices_`. However, the function originally contained separate checks for the length of `credIds_` against `amounts_` and `credIds_` against `maxPrices_`, leading to potential issues if the arrays are of unequal lengths but bypass one of these checks.

The root cause of this issue lies in the fact that the checks were conducted independently, potentially allowing certain cases of unequal array lengths to go unnoticed until an out-of-bounds array access occurs.

**Impact:**

If the arrays `credIds_`, `amounts_`, and `maxPrices_` are of different lengths, the function could perform unintended operations, resulting in potential out-of-bounds errors. This can cause unexpected behavior, including reversion of the transaction, or even leaving the contract in an unintended state if any operations were partially completed before the error was encountered.

**Proof of Concept:**

Consider the following scenario where `credIds_`, `amounts_`, and `maxPrices_` have different lengths:

```solidity
uint256[] memory credIds = new uint256[](2);
credIds[0] = 1;
credIds[1] = 2;

uint256[] memory amounts = new uint256[](1);
amounts[0] = 100;

uint256[] memory maxPrices = new uint256[](3);
maxPrices[0] = 500;
maxPrices[1] = 600;
maxPrices[2] = 700;

batchBuyShareCred(credIds, amounts, maxPrices, address(0));
```

Without the consolidated check, this could potentially pass the first check but fail later, leading to an unexpected out-of-bounds error during execution.

**Recommended Mitigation:**

Implement a consolidated length check at the beginning of the function to ensure all arrays are of equal length before proceeding with any operations:

```solidity
if (credIds_.length != amounts_.length || credIds_.length != maxPrices_.length) {
    revert InvalidArrayLength();
}
```

This modification ensures that the function will immediately revert if the arrays are not of equal length, preventing any unintended operations and preserving the integrity of the batch operation.

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L200
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L220