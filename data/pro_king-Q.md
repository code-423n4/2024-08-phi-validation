## Title: Allowing the Same Individual to Act as Both Artist and Referrer In `PhiRewards::depositRewards` Function Reduces Incentives for True Referrals Causes Decrement The Effeicvtiveness of Refferel System


## Description
In the `PhiRewards::depositRewards` function, the protocol allows the same individual to act as both the artist and the referrer. If the `referral_` address matches the `minter_` address, the function reallocates the referral reward to the artist, effectively nullifying the referral reward. This can reduce the effectiveness of the referral system by diminishing the incentive for genuine referrals, as the artist could claim both rewards without any external referral taking place.
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L97

## Impact
 Allowing the same individual to receive both the artist and referral rewards can weaken the intended purpose of the referral system, which is to encourage users to bring new participants to the platform. This reduction in incentivization may result in fewer genuine referrals, limiting the protocol's growth and outreach. It could also create an unfair advantage for artists who can game the system by self-referring, leading to an imbalance in the reward distribution.


## Recommended mitigation
 Implement a check to ensure that the `referral_` address is distinct from the `minter_` address. If the same individual is detected in both roles, consider either rejecting the transaction or providing a reduced reward for self-referral scenarios to maintain the integrity of the referral system. Alternatively, design a separate reward structure that fairly compensates different roles while preventing potential abuse.