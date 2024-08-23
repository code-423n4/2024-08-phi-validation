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
