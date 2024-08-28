Zero Amount Deposit Should Revert to Prevent Unnecessary Gas Consumption

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L68 
In the deposit function at Line 68, the case where the amount is zero should trigger a revert, as this leads to unnecessary gas consumption. The solution is to implement a check similar to the one at Line 91 in src/reward/CuratorRewardsDistributor.sol to ensure that the transaction is reverted if the amount is zero, thereby preventing the operation from proceeding.
MANUAL review