
1. `_addCredIdPerAddress` is public which can allow unauthorized manipulation. https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L685 

2. `removeCredIdPerAddress` is public which can allow unauthorized manipulation https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L695

3. The `getPositionsForCurator` function in the Cred contract uses incorrect indexing when populating return arrays, potentially leading to out-of-bounds access. https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L513
```solidity
solidityCopycredIds = new uint256[](stopIndex - start_);
amounts = new uint256[](stopIndex - start_);

for (uint256 i = start_; i < stopIndex; i++) {
    // ... 
    credIds[i] = credId;  // Incorrect: should use [index]
    amounts[i] = amount;  // Incorrect: should use [index]
    // ...
}
```

4. The `getShareNumber` function in the Cred contrac uses EnumerableMap.get(), causing it to revert for addresses with no balance instead of returning zero. This can lead to unexpected failures in systems relying on this function to check share balances. To fix, replace get() with tryGet().https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L427