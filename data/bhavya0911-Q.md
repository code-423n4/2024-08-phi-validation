| |Issue|Instances|
|-|:-|:-:|
| NC-1 | Redundant else-if | 1 |
| NC-2 | Redundant variable in calculation | 1 |
| NC-3 | Accessing same storage variable twice, instead of once | 1 |
| NC-4 | Redundant checks in `safeBatchTransferFrom()` function as `_beforeTokenTransfer` hook is not used | 1 |
| NC-5 | Adding zero-value error for public view functions | 4 |
| NC-6 | Redundant loop | 1 |
### [NC-1] Redundant else-if
Else if is used to check for a condition which will be true if the previous if condition is false, making it redundant check and should use else instead.
*Instances (1)*:
```solidity
File: /src/reward/PhiRewards.sol

99:        } else if (referral_ != address(0)) {

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L171)

### [NC-2] Redundant variable in calculation
There is a redundant variable in calculation. `distributeAmount` is defined by `totalBalance - royaltyfee`. Now, in line 120, directly using `totalBalance` would be cheaper to get the same result with `royaltyfee + distributeAmount` paying extra gas.

*Instances (1)*:
```solidity
File: /src/reward/CuratorRewardsDistributor.sol

99:        uint256 distributeAmount = totalBalance - royaltyfee;

120:         _msgSender().safeTransferETH(royaltyfee + distributeAmount - actualDistributeAmount);
```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L120)

### [NC-3] Accessing same storage variable twice, instead of once
Instead of reading from cold storage twice, store the value in a memory variable, pass it through the if and return if it satisfies it.

*Instances (1)*:
```solidity
File: /src/art/PhiNFT1155.sol
 242:       if (bytes(advancedTokenURI[tokenId_][minter_]).length > 0) {
            return advancedTokenURI[tokenId_][minter_];
        } 

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L242)

### [NC-4] Redundant checks in `safeBatchTransferFrom()` function as `_beforeTokenTransfer` hook is not used
Instead of using `_beforeTokenTransfer()` hook to add custom checks and logic, custom `safeBatchTransferFrom()` is implemented which has similar checks as the inherited one, which is being call at the end of the function.

*Instances (1)*:
```solidity
File: /src/art/PhiNFT1155.sol

332:    function safeBatchTransferFrom(
        address from_,
        address to_,
        uint256[] memory ids_,
        uint256[] memory values_,
        bytes memory data_
    )
        public
        override
    {
        for (uint256 i; i < ids_.length; i++) {
            if (from_ != address(0) && soulBounded(ids_[i])) revert TokenNotTransferable();
        }
        address sender = _msgSender();
        if (from_ != sender && !isApprovedForAll(from_, sender)) {
            revert ERC1155MissingApprovalForAll(sender, from_);
        }
        _safeBatchTransferFrom(from_, to_, ids_, values_, data_);
    }

    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {
        // This function is intentionally left empty to allow for upgrades
    }

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L323)

### [NC-5] Adding zero-value error for public view functions
Error for zero-value should be added to view functions, where zero/default value signifies that requested data has not been set yet, similar to ERC721 implementation of `ownerOf()` function.

*Instances (4)*:
```solidity
File: /src/Cred.sol

392:        return creds[credId_].creator;

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L392)

```solidity
File: /src/Cred.sol

396:        return creds[credId_].currentSupply;

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L396)

```solidity
File: /src/Cred.sol

400:        buyShareRoyalty = creds[credId_].buyShareRoyalty;

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L400)

```solidity
File: /src/Cred.sol

401:        sellShareRoyalty = creds[credId_].sellShareRoyalty;

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L401)

### [NC-6] Redundant loop
Two same parameter loops are placed one after the other. Logic placed in later loop in be incorporated in the first one itself.

*Instances (1)*:
```solidity
File: /src/Cred.sol
 751:        for (uint256 i = 0; i < credIds_.length; ++i) {
            uint256 credId = credIds_[i];
            uint256 amount = amounts_[i];

            _updateCuratorShareBalance(credId, curator, amount, isBuy);

            if (isBuy) {
                creds[credId].currentSupply += amount;
                lastTradeTimestamp[credId][curator] = block.timestamp;
            } else {
                if (block.timestamp <= lastTradeTimestamp[credId][curator] + SHARE_LOCK_PERIOD) {
                    revert ShareLockPeriodNotPassed(
                        block.timestamp, lastTradeTimestamp[credId][curator] + SHARE_LOCK_PERIOD
                    );
                }
                creds[credId].currentSupply -= amount;
            }

            creds[credId].latestActiveTimestamp = block.timestamp;
        }

        for (uint256 i = 0; i < credIds_.length; ++i) {
            uint256 credId = credIds_[i];

            protocolFeeDestination.safeTransferETH(protocolFees[i]);
            IPhiRewards(phiRewardsAddress).deposit{ value: creatorFees[i] }(
                creds[credId].creator, bytes4(keccak256("CREATOR_ROYALTY_FEE")), ""
            );
            emit Royalty(creds[credId].creator, credId, creatorFees[i]);

            emit Trade(curator, credId, isBuy, amounts_[i], prices[i], protocolFees[i], creds[credId].currentSupply);
        } 

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L751)

It can be combined in single loop in this way:

```solidity
751:        for (uint256 i = 0; i < credIds_.length; ++i) {
            uint256 credId = credIds_[i];
            uint256 amount = amounts_[i];

            _updateCuratorShareBalance(credId, curator, amount, isBuy);

            if (isBuy) {
                creds[credId].currentSupply += amount;
                lastTradeTimestamp[credId][curator] = block.timestamp;
            } else {
                if (block.timestamp <= lastTradeTimestamp[credId][curator] + SHARE_LOCK_PERIOD) {
                    revert ShareLockPeriodNotPassed(
                        block.timestamp, lastTradeTimestamp[credId][curator] + SHARE_LOCK_PERIOD
                    );
                }
                creds[credId].currentSupply -= amount;
            }

            creds[credId].latestActiveTimestamp = block.timestamp;

            protocolFeeDestination.safeTransferETH(protocolFees[i]);
            IPhiRewards(phiRewardsAddress).deposit{ value: creatorFees[i] }(
                creds[credId].creator, bytes4(keccak256("CREATOR_ROYALTY_FEE")), ""
            );
            emit Royalty(creds[credId].creator, credId, creatorFees[i]);

            emit Trade(curator, credId, isBuy, amounts_[i], prices[i], protocolFees[i], creds[credId].currentSupply);
        }
```
This reduced the gas cost of going through 2 different, but same loops but combining them into single.