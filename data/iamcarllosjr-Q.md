## ETH sent to PhiNFT1155.sol will be lost.

## Vulnerability Detail
The PhiNFT1155.sol contract has a receive() function, designed to receive ETH. However, there is no function in the contract to withdraw the ETH sent. Suppose the admin sends a large amount of ETH to the protocol, so that it sends the fee for the artwork created to protocolFeeDestination from the protocol balance. If the protocol has to be stopped, all the remaining ETH cannot be withdrawn. 

## Impact
It will not be possible to recover ETH sent to the contract. If the protocol needs to be paused, or even inactivated, the ETH in the contract cannot be withdrawn.

## Code Snippet
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L359

## Tool used

Manual Review

## Recommendation
Consider creating a function to withdraw the ETH sent to the contract.