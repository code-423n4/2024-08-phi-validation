## Gas Optimization of `handleRewardsAndGetValueSent`

### Summary

`handleRewardsAndGetValueSent` in [src/reward
/PhiRewards.sol](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L123C14-L123C42) does not include a check to validate if the `quantity_` is non-zero before executing the `depositRewards` function. 

This omission can lead to unnecessary gas consumption when users inadvertently or maliciously invoke the function with `quantity_ = 0`. Implementing a simple `require` check to ensure that `quantity_` is greater than zero can prevent these unnecessary operations and optimize gas usage.

### Details

In the function `handleRewardsAndGetValueSent`, the `quantity_` parameter is used to calculate reward distributions and to determine the total deposit amount. However, there is no validation to ensure that `quantity_` is greater than zero before these calculations and the subsequent execution of the `depositRewards` function.

Allowing a zero `quantity_` leads to the following issues:

- Unnecessary Gas Consumption: Even with `quantity_ = 0`, the function executes various calculations and state changes, consuming gas without any meaningful effect.
- Event Emission: The function might emit events even when `quantity_ = 0`, leading to unnecessary gas usage and event log clutter.

Suggested Fix: 

Add a require statement at the beginning of the `handleRewardsAndGetValueSent` function to ensure that `quantity_` is greater than zero:

```
function handleRewardsAndGetValueSent(
    uint256 artId_,
    uint256 credId_,
    uint256 quantity_,
    uint256 mintFee_,
    bytes calldata addressesData_,
    bool chainSync_
)
    external
    payable
{
    // Prevent zero quantity
    require(quantity_ > 0, "Quantity must be greater than 0");

    if (computeMintReward(quantity_, mintFee_) != msg.value) {
        revert InvalidDeposit();
    }

    depositRewards(
        artId_,
        credId_,
        addressesData_,
        quantity_ * (mintFee_ + artistReward),
        quantity_ * referralReward,
        quantity_ * verifierReward,
        quantity_ * curateReward,
        chainSync_
    );
}
```