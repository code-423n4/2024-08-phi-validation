This is the of the function which has some issues:-
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