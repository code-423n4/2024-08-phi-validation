# QA Report

| *Issue* | *Description*                                                                  |
|---------|--------------------------------------------------------------------------------|
| [L-01]  | No check for maxPrices array length                                            |

## [L-01] No check for maxPrices array length

In ```_validateAndCalculateBatch()``` in [Cred.sol](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L826), there is no check for whether the ```priceLimits_``` array input length matches the ```credIds_``` array length, like there is for ```amounts_```. A wrong ```priceLimits_``` array length currently only reverts further down the function with a rather vague ```panic: array out-of-bounds access (0x32)```, wasting gas for loop iterations and calculations, until the error is found.

### **Recommended Mitigation Steps**

Add a check for matching array length, reverting with ```InvalidArrayLength()```.

```diff
uint256 length = credIds_.length;
-   if (length != amounts_.length) {
+   if (length != amounts_.length || length != priceLimits_.length) {
        revert InvalidArrayLength();
    }
    if (length == 0) {
        revert EmptyBatchOperation();
    }
```
