# `PhiFactory::withdraw` cannot be called because lack of receive/fallback on the contract

## Link to github

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L786-L789

## Summary

The `withdraw` function is called whenever owner wants to withdraw ether from the contract. Considering the design of the `PhiFactory` contract is not supposed to receive/hold ether and lack of receive/fallback function making `withdraw` function redundant or unusable.

## Tools Used

Manual review

## Recommendation

There are two recommendations:

1. Remove the withdraw Function: If the contract is not supposed to handle ether, the withdraw function should be removed to avoid confusion and potential security issues.
2. Implement receive or fallback Function: If there is a possibility of receiving ether (even accidentally), implementing a minimal receive or fallback function would allow the contract to handle and withdraw such ether.