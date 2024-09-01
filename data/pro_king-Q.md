## [QA-1] Title: Allowing the Same Individual to Act as Both Artist and Referrer In `PhiRewards::depositRewards` Function Reduces Incentives for True Referrals Causes Decrement The Effeicvtiveness of Refferel System


## Description
In the `PhiRewards::depositRewards` function, the protocol allows the same individual to act as both the artist and the referrer. If the `referral_` address matches the `minter_` address, the function reallocates the referral reward to the artist, effectively nullifying the referral reward. This can reduce the effectiveness of the referral system by diminishing the incentive for genuine referrals, as the artist could claim both rewards without any external referral taking place.
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L97

## Impact
 Allowing the same individual to receive both the artist and referral rewards can weaken the intended purpose of the referral system, which is to encourage users to bring new participants to the platform. This reduction in incentivization may result in fewer genuine referrals, limiting the protocol's growth and outreach. It could also create an unfair advantage for artists who can game the system by self-referring, leading to an imbalance in the reward distribution.


## Recommended mitigation
 Implement a check to ensure that the `referral_` address is distinct from the `minter_` address. If the same individual is detected in both roles, consider either rejecting the transaction or providing a reduced reward for self-referral scenarios to maintain the integrity of the referral system. Alternatively, design a separate reward structure that fairly compensates different roles while preventing potential abuse.




## [QA-2] Title: Fairness Violation due to Royalty Fees and Rounding Difference Reducing From Curator Rewards in `CuratorRewardsDistributor::distribute` Function


## Description
In the `CuratorRewardsDistributor::distribute` function, curators receive less than their expected share due to two factors: royalty fees and rounding difference. The protocol subtracts a royalty fee from the total balance before distribution, and rounding difference is also subtracted from the curators rewards. This results in curators potentially receiving smaller rewards than they anticipate, leading to dissatisfaction or reduced engagement with the platform.
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L120

## Impact
 Curators may perceive the reward distribution as unfair due to losses from royalty fees and rounding differences. This could result in reduced participation and engagement with the platform, which may harm the platform’s reputation and long-term success. Over time, curators may become demotivated if they feel the system undercompensates them consistently.


## Recommended mitigation
1. Separate Rounding Compensation: Introduce a mechanism that compensates curators for rounding discrepancies by either rounding up or allocating the leftover amounts proportionally to ensure curators are not penalized by the distribution process.
2. Royalty Fee Adjustment: Consider covering the royalty fees from the protocol itself rather than deducting them from the curator’s distribution balance. Alternatively, introduce a clear and transparent fee structure so curators understand how their rewards are impacted and ensure they are not unfairly penalized by this deduction.
3. Fee Transparency: Clearly communicate to curators how the royalty fees and rounding are applied to their rewards, and ensure that any deductions are justified and accounted for in a transparent manner.




## [QA-3] Title:Lack of Length Validation for Input Arrays  In `Cred::getBatchBuyPrice`, `Cred::getBatchSellPrice` and `PhiFactory::getTotalMintFee` Functions Causes Mismatched Operations



## Description
The functions `Cred::getBatchBuyPrice`, `Cred::getBatchSellPrice`, and `PhiFactory::getTotalMintFee` do not validate whether the lengths of the input arrays `credIds_` and `amounts_` (or `artId_` and `quantitys_` in `PhiFactory::getTotalMintFee` ) are the same before iterating through them. This can result in mismatched operations where one array is shorter than the other, leading to incorrect calculations or potential out-of-bounds errors.
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L343
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L364
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L517

## Impact
1. Incorrect Price/Fee Calculation: If the lengths of the input arrays are not equal, this could result in incorrect buy/sell price or mint fee calculations, leading to financial inaccuracies or incorrect user balances.
2. Potential Reverts or Undesired Behavior: Mismatched lengths could cause out-of-bounds access to the smaller array, leading to transaction reverts, or even worse, unhandled behavior that could impact user trust or protocol stability


## Recommended mitigation
Add explicit checks to ensure that the lengths of the input arrays `credIds_` and `amounts_` (or `artId_` and `quantitys_`) are equal before processing them. If the lengths do not match, the function should revert with an appropriate error message.

In `Cred::getBatchBuyPrice` function:

```diff
    function getBatchBuyPrice(
        uint256[] calldata credIds_,
        uint256[] calldata amounts_
    )
        external
        view
        returns (uint256)
    {
+        require(credIds_.length == amounts_.length, "Input array lengths mismatch");
        uint256 total;
        for (uint256 i = 0; i < credIds_.length; ++i) {
            total += IBondingCurve(creds[credIds_[i]].bondingCurve).getBuyPriceAfterFee(
                credIds_[i], creds[credIds_[i]].currentSupply, amounts_[i]
            );
        }
        return total;
    }
```
In `Cred::getBatchSellPrice` function:
```diff
function getBatchSellPrice(
        uint256[] calldata credIds_,
        uint256[] calldata amounts_
    )
        external
        view
        returns (uint256)
    {
+        require(credIds_.length == amounts_.length, "Input array lengths mismatch");
       
        uint256 total;
        for (uint256 i = 0; i < credIds_.length; ++i) {
            total += IBondingCurve(creds[credIds_[i]].bondingCurve).getSellPriceAfterFee(
                credIds_[i], creds[credIds_[i]].currentSupply, amounts_[i]
            );
        }
        return total;
    }

```

In `PhiFactory::getTotalMintFee` function:

```diff
    function getTotalMintFee(
        uint256[] calldata artId_,
        uint256[] calldata quantitys_
    )
        external
        view
        returns (uint256)
    {
+     require(artId_.length == quantitys_.length, "Input array lengths mismatch");
        uint256 totalMintFee;
        for (uint256 i = 0; i < artId_.length; i++) {
            uint256 artId = artId_[i];
            totalMintFee = totalMintFee + getArtMintFee(artId, quantitys_[i]);
        }
        return totalMintFee;
    }
```

