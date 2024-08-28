this might not be a critical issue tho but it is good to do follow best practice to avoid some future issue

https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L786

function withdraw() external onlyOwner {
        protocolFeeDestination.safeTransferETH(address(this).balance);
    }

 Adding a balance check makes the function more robust and user-friendly by handling cases where there are no funds to withdraw.
