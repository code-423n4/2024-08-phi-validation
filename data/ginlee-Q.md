# [L-01] Insufficient Payment Validation in PhiRewards::handleRewardsAndGetValueSent

function handleRewardsAndGetValueSent originally included a strict equality check (==) to compare msg.value with the calculated reward amount using computeMintReward. This strict check could cause the transaction to fail if a user mistakenly sends more ETH than required. 

## Detailed Code
```solidity
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
@>      if (computeMintReward(quantity_, mintFee_) != msg.value) {
            revert InvalidDeposit();
        } 
```

## Mitigation
Modify the handleRewardsAndGetValueSent function as suggested to check for sufficient fund and refund any excess. This ensures smoother user interactions and prevents unnecessary transaction failures.
```solidity
if (computeMintReward(quantity_, mintFee_) > msg.value) {
            revert InvalidDeposit();
        }
```