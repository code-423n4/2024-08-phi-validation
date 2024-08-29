## [01]Cred.sol :: initialize() does not check if phiSignerAddress_ is set to address(0)
[initialize()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L69-L93) is used for setting all the values of the state variables. However, unlike the **`setPhiSignerAddress()`**, it does not check whether **`phiSignerAddress_`** is not address(0).
```Solidity
function setPhiSignerAddress(address phiSignerAddress_) external nonZeroAddress(phiSignerAddress_) onlyOwner {
        phiSignerAddress = phiSignerAddress_;
        emit PhiSignerAddressSet(phiSignerAddress_);
    }
```
This can be a serious problem because if **`phiSignerAddress_`** is initially set to address(0), all signatures in **`createCred()`** and **`updateCred()`** will be incorrectly validated. This happens because **`ECDSA.recover()`** returns address(0) when a signature is invalid, making every signature appear valid.

To prevent this, ensure that **`initialize()`** checks that **`phiSignerAddress_`** is not set to address(0)

## [02] PhiNFT1155.sol :: credChainId cannot be updated, a hard fork would prevent curatorRewardsDistributor.deposit from ever being called.
The **`credChainId`** is set in the [initialize()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L113), but there's no function available to update it later. In the event of a hard fork, where **`block.chainid`** changes, **`credChainId`** would no longer match the new chainid. This mismatch would prevent **`chainSync_`** from ever being true, which in turn would stop [curatorRewardsDistributor.deposit](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L108-L110) from being called.

To address this, implement a function like **`setCredChainId()`** to allow updating **`credChainId`** in the event of a hard fork.

## [03]PhiNFT1155.sol:: Royalties can be set above the ROYALTY_BPS_TO_PERCENT.
[updateRoyalties()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L195-L203) calls [_updateRoyalties()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/CreatorRoyaltiesControl.sol#L58-L66) and allows the owner of an NFT to set the royalty percentage. However, the issue is that it does not check whether the specified royalties exceed the maximum allowable limit.
```Solidity
function royaltyInfo(
        uint256 tokenId,
        uint256 salePrice
    )
        public
        view
        returns (address receiver, uint256 royaltyAmount)
    {
        RoyaltyConfiguration memory config = getRoyalties(tokenId);
@>      royaltyAmount = (config.royaltyBPS * salePrice) / ROYALTY_BPS_TO_PERCENT;
        receiver = config.royaltyRecipient;
    }

```
When royaltyAmount is calculated, it is divided by [ROYALTY_BPS_TO_PERCENT](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/CreatorRoyaltiesControl.sol#L11), which in this case is 10,000 (the maximum value).

To resolve this, implement a check in **`updateRoyalties()`** to ensure that the value does not exceed **`ROYALTY_BPS_TO_PERCENT`**.