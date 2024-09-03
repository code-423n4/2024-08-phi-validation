# [01] Typographical errors

## Description 

In line [584](https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L584) from the Cred.sol file it states "amount_ The amount to buy or se;;" instead of "amount_ The amount to buy or sell"

## [02] Artist can bypass the endTime conditions

https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L233-L238
https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L583-L587
## Description 
Artist, after creating the token, can bypass the endTime conditions. When a new NFT token is created, the `_validateArtCreation` function is called, which checks `endTime` and `artAddress` against address(0)
```solidity
function _validateArtCreation(ERC1155Data memory createData_) private view {


if (arts[createData_.artId].artAddress != address(0)) revert ArtAlreadyCreated(); 

if (createData_.endTime <= block.timestamp) revert EndTimeInPast();

if (createData_.endTime <= createData_.startTime) revert EndTimeLessThanOrEqualToStartTime();

}
```
After creation, the artist can call the `updateArtSettings` function and change the `endTime`, namely make it equal to `startTime` and equal to `block.timestamp` 

```solidity
 function updateArtSettings(
        uint256 artId_,
        string memory url_,
        address receiver_,
        uint256 maxSupply_,
        uint256 mintFee_,
        uint256 startTime_,
        uint256 endTime_,
        bool soulBounded_,
        IPhiNFT1155Ownable.RoyaltyConfiguration memory configuration
    )
        external
        onlyArtCreator(artId_)
    {
        if (receiver_ == address(0)) {
            revert InvalidAddressZero();
        }

->       if (endTime_ < startTime_) {
            revert InvalidTimeRange();
        }
->        if (endTime_ < block.timestamp) {
            revert EndTimeInPast();
        }
        ...}
```
## Recommendation 
A stricter equality should be made 
```diff
- if (endTime_ < startTime_) {
+ if (endTime_ <= startTime_) {
            revert InvalidTimeRange();
        }
- if (endTime_ < block.timestamp) {
+ if (endTime_ <= block.timestamp) {
            revert EndTimeInPast();
```

# [03] Open Todos

## Description 
Open To-dos can point to architecture or programming issues that still need to be resolved. Often these kinds of comments indicate areas of complexity or confusion for developers. This provides value and insight to an attacker who aims to cause damage to the protocol

### Code location
Line [474](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L474) PhiFactory.sol 
Line [552](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L552) PhiFactory.sol

# [04] wrong modifier name
https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L77-L82
## Description 
In the contract PhiNFT1155.sol has an `onlyArtCreator` [modifier](https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L77-L82) that checks that `msg.sender == artist || msg.sender == owner()`

```solidity
modifier onlyArtCreator(uint256 tokenId_) {

uint256 artId = _tokenIdToArtId[tokenId_];

address artist = phiFactoryContract.artData(artId).artist;

if (msg.sender != artist && msg.sender != owner()) revert NotArtCreator();

_;

}
```

The problem is that the name does not correspond to what happens in the modifier, the name suggests that the function that has this modifier can be called only by the **artist**, but in fact it can be called by the **owner** of the contract as well. This can be confusing for users.

## Recommendation 
The modifier name should be changed
```diff
- modifier onlyArtCreator(uint256 tokenId_) {...}
+ modifier onlyArtCreatorAndOwner(uint256 tokenId_) {...}
```