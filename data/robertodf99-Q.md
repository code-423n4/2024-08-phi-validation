# Accrual of map keys in `shareBalance` will result in expensive curator rewards distribution in the long term
## Impact
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L77-L130
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L666-L682

Every time a curator buys a share of a cred, they are added to the mapping `shareBalance`:

```javascript
shareBalance[credId_].set(sender_, currentNum + amount_);
```

When they sell the cred, instead of removing them from the mapping, it is set to 0:

```javascript
shareBalance[credId_].set(sender_, currentNum - amount_);
```

The `CuratorRewardsDistributor::distribute` function is responsible for distributing rewards to curators. It retrieves curator data using the `Cred::_getCuratorData` function, which iterates over all elements in the mapping and filters out non-shareholders (i.e., those with 0 shares):

```javascript
...
for (uint256 i = start_; i < stopIndex; ++i) {
    (address key, uint256 shareAmount) = shareBalance[credId_].at(i);

    if (isShareHolder(credId_, key)) {
        result[index] = CuratorData(key, shareAmount);
        index++;
    }
}
...
```
While this design is functional, it poses long-term sustainability challenges for the protocol. As the number of users grows, the frequency of curator changes will increase. When the mapping becomes very large, the gas cost of iterating over its length may exceed the combined value of the distribution reward and the largest shareholder’s reward. Without sufficient economic incentives to distribute rewards, funds could become trapped in the contract.

## Proof of Concept
The impact of this design can be demonstrated by simulating a scenario where users repeatedly buy and sell a credential. After running the simulation with 1,000 iterations (restricted to avoid `EvmError: OutOfGas`), the gas used during distribution is calculated. Running `forge test --mt testDoSDistribution --gas-report` you get the following results:

| src/Cred.sol:Cred contract |                 |          |          |          |         |
|----------------------------|-----------------|----------|----------|----------|---------|
| Deployment Cost            | Deployment Size |          |          |          |         |
| 5114273                    | 23770           |          |          |          |         |
| Function Name              | min             | avg      | median   | max      | # calls |
| buyShareCred               | 251002          | 251010   | 251002   | 268102   | 2000    |
| createCred                 | 452445          | 452445   | 452445   | 452445   | 1       |
| getCreatorRoyalty          | 654             | 654      | 654      | 2654     | 8001    |
| **getCuratorAddresses**    | **15536032**    | **15536032** | **15536032** | **15536032** | **1**       |
| getShareNumber             | 900             | 900      | 900      | 900      | 2       |
| initialize                 | 209081          | 209081   | 209081   | 209081   | 1       |
| isExist                    | 557             | 558      | 557      | 2557     | 4003    |
| protocolFeePercent         | 417             | 1417     | 2417     | 2417     | 8002    |
| sellShareCred              | 134807          | 134807   | 134807   | 134807   | 2000    |

The results indicate that distributing rewards could cost at least, approximately 1.5 million gas units for every 1,000 curators interacting with a particular cred, with costs expected to scale linearly as the number of curators increases.

```javascript
function testDoSDistribution() public {
    uint256 credId = 1;
    uint256 depositAmount = 1 ether;

    // Deposit some ETH to the curatorRewardsDistributor
    curatorRewardsDistributor.deposit{value: depositAmount}(
        credId,
        depositAmount
    );
    uint256 nIter = 2000;
    address user;

    // Signal creds for different users
    for (uint256 i = 1; i <= nIter; i++) {
        user = makeAddr(string(abi.encodePacked("user", i)));
        vm.startPrank(user);
        vm.deal(user, bondingCurve.getBuyPriceAfterFee(credId, 1, 1));
        cred.buyShareCred{
            value: bondingCurve.getBuyPriceAfterFee(credId, 1, 1)
        }(credId, 1, 0);
        vm.warp(block.timestamp + 10 minutes + 1);
        cred.sellShareCred(credId, 1, 0);
        vm.stopPrank();
    }

    // Distribute rewards
    vm.prank(owner);
    uint256 gasBefore = gasleft();
    curatorRewardsDistributor.distribute(credId);
    console2.log("Gas consumed:", gasBefore - gasleft());
    assertEq(phiRewards.balanceOf(user), 0);
}
```
## Tools Used
Manual review.

## Recommended Mitigation Steps
Use the remove method in `Cred::_updateCuratorShareBalance` when the amount a user owns is reduced to 0:

```diff
function _updateCuratorShareBalance(
    uint256 credId_,
    address sender_,
    uint256 amount_,
    bool isBuy
) internal {
    (, uint256 currentNum) = shareBalance[credId_].tryGet(sender_);

    if (isBuy) {
        if (currentNum == 0 && !_credIdExistsPerAddress[sender_][credId_]) {
            _addCredIdPerAddress(credId_, sender_);
            _credIdExistsPerAddress[sender_][credId_] = true;
        }
        shareBalance[credId_].set(sender_, currentNum + amount_);
    } else {
+       shareBalance[credId_].set(sender_, currentNum - amount_);
        if ((currentNum - amount_) == 0) {
            _removeCredIdPerAddress(credId_, sender_);
            _credIdExistsPerAddress[sender_][credId_] = false;
+           shareBalance[credId_].remove(sender_);
        }
-       shareBalance[credId_].set(sender_, currentNum - amount_);
    }
}
```