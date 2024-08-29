This is the function which has some issues:-
  https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/Claimable.sol#L67-L84
## [M] The `Claimable::_decodeMerkleClaimData` heler function has some invulnerability realated with insuffificent input validation of `msg.data` length and the decode structure.

**Description:**

- The `Claimable::_decodeMerkleClaimData` function checks `length` which is not feasible way to check the length.
  `if (msg.data.length < 260) revert InvalidMerkleClaimData` this line is not enough to check the correct length of the
  data.
- The function does not check data dynamically, it just expect the data should be less than `260` which not make any
  sense, input data might be courrpted or invalid.

**Impact:**

- This could allow invalid or malicious data to be processed by the contract, leading to unexpected behavior which may
  affect process claims.
- This may call to be reverted or any unexpected behavior.

**Proof of Concept:**

```javascript
uint256 offset = 4;
uint256 proofLength;
        assembly {
            proofLength := mload(add(msg.data, 68)) // Load proof length from msg.data
        }
        uint256 expectedLength = offset + 20 + 32 * proofLength + 20 + 32 + 32 + 32 + 32 + 4 + bytes(imageURI).length;

```

Above code can be the one way to implement the dynamic length check Or you can do, First `Decode` the fixed data then
extract the proof from the data using assembly code ,then you can concat the data and check the length of the data and
compare the decode the dynamic data part. `This might be risky as well as tricky`.

**Recommended Mitigation:**

- Implement the dynamic length check for the data.
- Properly validate the decoded data.
- Use the `assembly` code to extract the data and validate the data.


This is the function which has some issues:-
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/Claimable.sol#L22-L44

## The `Claimable::signatureClaim` function in the contract lacks critical input validation checks, which could lead to several security vulnerabilities.

**Description:** There are various issue like:

1. Address Checking - `ref_` `verifier_` `minter_` these parameters are not checked for zero address input which might
   cause attacker send eth to zero address.
2. ChainId Checking - `chainId_` is not checked, which means attacker can pass any random value which might cause the
   contract to behave unexpectedly or break it.
3. `expiresIn_` false parameter can be pass which might hold the funds for longer period of time which is not expected.
4. `imageURI_` is not checked for empty string which might cause the contract to behave unexpectedly or break it.

**Impact:** 1. Loss of Funds. 2. Replay Attacks. 3. Inconsistent Data.

Proof of Concept: An attacker could call the signatureClaim function with address(0) for minter*, ref*, and verifier*, a
quantity* of 0, and an empty imageURI\_. This could lead to unintended behavior such as burning tokens or assets,
inconsistent data processing, or even replaying the transaction on different chains.

**Recommended Mitigation:** Follow the best practices for input validation and explicitly check for zero addresses,
empty strings, and invalid values. Ensure that the function is not vulnerable to replay attacks and that the data is
consistent across all chains.

```javascript
if (minter_ == address(0) || ref_ == address(0) || verifier_ == address(0)) {
  revert("Claimable: zero address");
}
if (chainId_ == expectedChainId) {
  revert("Claimable: zero chainId");
}
if (expiresIn_ == false) {
  revert("Claimable: expiresIn_ is false");
}
if (bytes(imageURI_).length == 0) {
  revert("Claimable: empty imageURI_");
}
```