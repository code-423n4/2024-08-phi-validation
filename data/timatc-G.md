The two loops in the _executeBatchTrade (https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L734-L783) can be merged to save gas: 

    function _executeBatchTrade(
    uint256[] calldata credIds_,
    uint256[] calldata amounts_,
    address curator,
    uint256[] memory prices,
    uint256[] memory protocolFees,
    uint256[] memory creatorFees,
    bool isBuy
    )
    internal
    whenNotPaused
    nonReentrant
    {
    if (curator == address(0)) {
        revert InvalidAddressZero();
    }

    for (uint256 i = 0; i < credIds_.length; ++i) {
        uint256 credId = credIds_[i];
        uint256 amount = amounts_[i];

        _updateCuratorShareBalance(credId, curator, amount, isBuy);

        if (isBuy) {
            creds[credId].currentSupply += amount;
            lastTradeTimestamp[credId][curator] = block.timestamp;
        } else {
            if (block.timestamp <= lastTradeTimestamp[credId][curator] +                  SHARE_LOCK_PERIOD) {
                revert ShareLockPeriodNotPassed(
                    block.timestamp, lastTradeTimestamp[credId][curator] + SHARE_LOCK_PERIOD
                );
            }
            creds[credId].currentSupply -= amount;
        }

        creds[credId].latestActiveTimestamp = block.timestamp;

        // Execute the operations from the second loop
        protocolFeeDestination.safeTransferETH(protocolFees[i]);
        IPhiRewards(phiRewardsAddress).deposit{ value: creatorFees[i] }(
            creds[credId].creator, bytes4(keccak256("CREATOR_ROYALTY_FEE")), ""
        );
        emit Royalty(creds[credId].creator, credId, creatorFees[i]);

        emit Trade(curator, credId, isBuy, amount, prices[i], protocolFees[i],       creds[credId].currentSupply);
    }
    }
