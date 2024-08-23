# [L-01] wrong assumption of block time in certain chains
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L33
the project states the lock period for a trade to be 10mins
`uint256 public constant SHARE_LOCK_PERIOD = 10 minutes;`
So for doing a sell, particular trade for a curator has to be greater than 10min
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L178-188
```
 if (block.timestamp <= lastTradeTimestamp[credId_][curator_] + SHARE_LOCK_PERIOD) {
                revert ShareLockPeriodNotPassed(
                    block.timestamp, lastTradeTimestamp[credId_][curator_] + SHARE_LOCK_PERIOD
                );
```
So in some chains these limits will be set shorter than expected.
Like Berachain, where protocol is going to be deployed has 5s block time according to its [documentation](https://docs.berachain.com/faq/#how-well-does-berachain-perform).
## Recommendation
Set the time limits with proper validation accordingly to all the chains the codebase is deployed

# [L-02] Signature reuse across different chains
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L146
Because the chain ID is not included in the `structhash`, all signatures are also valid when the project is launched on a chain with another chain ID. For instance, when it is also launched on Polygon. An attacker can now use all of the Ethereum signatures there. Because the Polygon addresses of user's (and potentially contracts, when the nonces for creating are the same) are often identical.so users can use other users signature to withdraw their funds on another chain
```

        bytes32 structHash = keccak256(abi.encode(WITHDRAW_TYPEHASH, from, to, amount, nonce, deadline));
        bytes32 digest = _hashTypedData(structHash);
        return SignatureCheckerLib.isValidSignatureNowCalldata(from, digest, sig);
    }
```
## Recommendation
Include The chain ID in the signature hash