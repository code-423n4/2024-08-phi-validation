## Title: Repeated Storage Access in For Loop Increases Gas Consumption In `CuratorRewardsDistributor::distribute` Function


## Description
In the `CuratorRewardsDistributor::distribute` function, curators receive less than their expected share due to two factors: royalty fees and rounding difference. The protocol subtracts a royalty fee from the total balance before distribution, and rounding difference is also subtracted from the curators rewards. This results in curators potentially receiving smaller rewards than they anticipate, leading to dissatisfaction or reduced engagement with the platform.
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L87
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L106


## Impact
 Repeated storage access can significantly increase gas costs, especially when the `distributeAddresses` array is large. This leads to inefficiencies in contract execution and higher transaction fees for users, especially during high-traffic periods or when interacting with a large dataset. Optimizing the gas usage by storing the length of the array in a local variable can result in a more efficient contract, benefiting both the contract and its users.


## Recommended mitigation
 Optimizing the gas usage by storing the length of the array in a local variable can result in a more efficient contract, benefiting both the contract and its users.

```diff
+      uint256 distributeAddressesLength = distributeAddresses.length;
+   for (uint256 i = 0; i < distributeAddresses.length; i++) {
-   for (uint256 i = 0; i < distributeAddresses.length; i++) {
            totalNum += credContract.getShareNumber(credId, distributeAddresses[i]);
        }


+    for (uint256 i = 0; i < distributeAddressesLength; i++) {
-    for (uint256 i = 0; i < distributeAddresses.length; i++) {

    // ...
}
