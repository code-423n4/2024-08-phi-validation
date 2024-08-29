Claimable.sol

1) Reentrancy Protection

The signatureClaim and merkleClaim functions involve external calls to the phiFactoryContract without any reentrancy guard (nonReentrant). Consider adding reentrancy protection if there are concerns about the trustworthiness of the IPhiFactory contract.

2) Lack of Return Values

The functions signatureClaim and merkleClaim do not have any return values, nor do they check for the success of the calls to phiFactoryContract.signatureClaim and phiFactory.merkleClaim. If these external calls fail, the functions will revert.

3)Visibility of _decodeMerkleClaimData Function

The _decodeMerkleClaimData function is marked as private. While this might be intentional for encapsulation, if there's a need for flexibility or testing, making it internal might be more appropriate. This would allow inherited contracts to access and override this function if needed.

CreatorRoyaltiesControl.sol

1) Typo in State Variable Name
initilaized is misspelled. It should be initialized. This typo can lead to confusion and potential bugs. It should be corrected for consistency and clarity.

Line 19 if (initilaized) revert AlreadyInitialized();
Line 20        royaltyRecipient = _royaltyRecipient;
Line 21        initilaized = true;

RewardControl.sol

1)In the _withdraw function, the amount parameter can be checked against balanceOf[from] and then subtracted. This should be safer.

PhiNFT1155.sol

1)mint Function

The mint function sets the minted status of an address but does not revert or handle cases where minting might fail. If mint fails or is called with invalid parameters, it might leave the minted status in an inconsistent state. Ensure that the mint function has appropriate checks and error handling. Consider checking the result of _mint.

2)claimFromFactory Function

The claimFromFactory function handles rewards and value transfers but does not ensure that the art ID is valid before processing. If the art ID is invalid or the mintFee is not set correctly, the function might behave unexpectedly.

Before proceeding with any logic, ensure that the art ID is valid and corresponds to an existing token ID. Add a validation check to verify that the art ID is indeed mapped to a valid token ID.

uint256 tokenId_ = _artIdToTokenId[artId_];
if (tokenId_ == 0) {
    revert("Invalid art ID");
}

Ensure that the mint fee is correctly set and matches the expected value. Retrieve the mint fee from the PhiFactory and verify that it is greater than zero and aligns with the expected fee structure.

uint256 mintFee = phiFactoryContract.artData(_tokenIdToArtId[tokenId_]).mintFee;
if (mintFee == 0) {
    revert("Invalid mint fee");
}
if (msg.value < mintFee) {
    revert("Insufficient funds sent");
}

BondingCurve.sol

1) Use of Ownable Instead of Ownable2Step

You are inheriting from Ownable2Step but initializing with Ownable in the constructor. This inconsistency might lead to confusion or unexpected behavior regarding ownership management.

2)Potential Division by Zero in _curve Function

The _curve function performs a division operation which could potentially lead to a division by zero if TOTAL_SUPPLY_FACTOR - targetAmount_ is zero. This could happen if targetAmount_ is set to TOTAL_SUPPLY_FACTOR.  Add a check to prevent division by zero.

3)Inconsistent Units

Ensure that units are consistently used throughout the contract, especially when dealing with ether and other currency values. This prevents issues related to unit conversions.

For example: 

function _getProtocolFee(uint256 price_) internal view returns (uint256) {
    return price_ * credContract.protocolFeePercent() / RATIO_BASE;
}

function _getCreatorFee(uint256 credId_, uint256 supply_, uint256 price_, bool isSign_) internal view returns (uint256 creatorFee) {
    if (!credContract.isExist(credId_)) {
        return 0;
    }
    (uint16 buyShareRoyalty, uint16 sellShareRoyalty) = credContract.getCreatorRoyalty(credId_);
    uint16 royaltyRate = isSign_ ? buyShareRoyalty : sellShareRoyalty;
    creatorFee = (price_ * royaltyRate) / RATIO_BASE;
}

If credContract.protocolFeePercent() or credContract.getCreatorRoyalty() returns values not in the same scale as RATIO_BASE, this could cause incorrect fee calculations. Ensure that all percentages are correctly scaled relative to RATIO_BASE.

Another example: 

function getPrice(uint256 supply_, uint256 amount_) public pure returns (uint256) {
    return _curve((supply_ + amount_) * 1 ether) - _curve(supply_ * 1 ether);
}

The use of 1 ether in this function assumes that supply_ and amount_ are scaled to match 1 ether


CuratorRewardsDistributor.sol


1)Incorrect ETH Transfer Calculation

The ETH amount transferred might be incorrect due to rounding errors in actualDistributeAmount. The calculation of the amount to send to _msgSender() might not match the actual amount required.


2)The loop in distribute could consume a lot of gas if the number of addresses is large.

for (uint256 i = 0; i < distributeAddresses.length; i++) {
    totalNum += credContract.getShareNumber(credId, distributeAddresses[i]);
}

Consider optimizing or limiting the number of addresses that can be processed in a single transaction. Alternatively, implement off-chain processing if dealing with a large number of addresses.


PhiRewards.sol

1)Potential Overflows

Calculations in depositRewards and handleRewardsAndGetValueSent could potentially lead to overflow or underflow if not handled properly.

Use SafeMath or similar libraries to handle arithmetic safely and avoid overflow issues, especially when dealing with reward calculations.

2)ETH Calculation Mismatch

The calculation of rewards in handleRewardsAndGetValueSent should match the msg.value provided by the user. Any mismatch might result in unexpected behavior.

Ensure that the computation of rewards (computeMintReward) correctly reflects the expected amount of ETH to be sent. Verify that the msg.value is handled properly and matches the computed reward amount.