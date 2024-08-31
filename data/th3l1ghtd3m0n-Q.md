# [L-1] Missing refund of excess mint fee in `PhiFactory::batchClaim()`
## Description

`PhiFactory::batchClaim()` fails to refund any excess of ETH that is paid by users that want to claim multiple art rewards:

```solidity
    function batchClaim(bytes[] calldata encodeDatas_, uint256[] calldata ethValue_) external payable {
        if (encodeDatas_.length != ethValue_.length) revert ArrayLengthMismatch();

        // calc claim fee
        uint256 totalEthValue;
        for (uint256 i = 0; i < encodeDatas_.length; i++) {
            totalEthValue = totalEthValue + ethValue_[i];
        }
        if (msg.value != totalEthValue) revert InvalidEthValue();

        for (uint256 i = 0; i < encodeDatas_.length; i++) {
            this.claim{ value: ethValue_[i] }(encodeDatas_[i]);
        }
    }
```

`batchClaim()` calls `claim()` function, which don't checks the amount of ether received. `claim()` function self calculates the amount of fee to call whether `merkleClaim()` or `signatureClaim()` functions:

```solidity
__ SNIP __
            uint256 mintFee = getArtMintFee(artId, quantity_);
            this.merkleClaim{ value: mintFee }(proof, claimData, mintArgs, leafPart_);
__ SNIP __
            uint256 mintFee = getArtMintFee(artId, quantity_);
            this.signatureClaim{ value: mintFee }(proof, claimData, mintArgs, leafPart_);
```

## Recommendation

The `claim()` function should use `msg.value` instead of calculating the fee internally when calling `merkleClaim()` or `signatureClaim()`. 

# [L-2] Missing refund of excess deposit rewards in `PhiRewards::handleRewardsAndGetValueSent()`
## Description
`PhiRewards::handleRewardsAndGetValueSent()` fails to refund any excess of ETH deposited as rewards allowing ether to stuck into the contract, since balances are calculated internally according to each role:

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


## Recommendation

Update to refund any excess deposited after updating balances.