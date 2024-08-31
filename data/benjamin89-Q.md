###
The duplicate cred id check can be updated using map.

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L848-L852

###

    mapping(uint256 => bool) memory seenCredIds;
    for (uint256 i = 0; i < length; ++i) {
        uint256 credId = credIds_[i];
        ... ...

        if (seenCredIds[credId]) {
            revert DuplicateCredId();
        }
        seenCredIds[credId] = true;
        ... ...
    }
