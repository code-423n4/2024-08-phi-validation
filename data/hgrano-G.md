## Gas Optimization Report

### G-01: Variable packing

The struct types from `IPhiFactory` should be packed using smaller variable types, particularly those stored
on the `PhiFactory`. Many of the variables are time stamps or counters which can be reduced. The
`IPhiFactory` also uses the `string` data type to represent the `verificationType`. This represents unnecessary
overhead to store string length and contents.

#### Recommendation

Change the data types of the following variables:
- `startTime`, `endTime` on `PhiArt` (to `uint64` for example).
- `verificationType` can be changed to `uint256` or an enum.

### G-02: Storage variable caching in PhiNFT1155::createArtFromFactory

The variable `tokenIdCounter` should be assigned to a local variable before being read elsewhere in the function:
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L139C35-L139C49

#### Recommendation

`tokenIdCounter` should cached as a local variable at the beginning of the function, then the `tokenIdCounter` itself need not be read again. To do the
update on line 149 we must use the storage variable again, however.

### G-03: Significant inefficiency in function call to retrieve aristRewardReceiver in PhiNFT1155

Code in question: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L183

The `PhiNFT1155` makes the above function call to retrieve the receiver of the `artId`. However, the implementation of the `artData` function is extremely
gas-heavy, reading many other state variables that are not used in this example.

#### Recommendation

The `aristRewardReceiver` could be passed in as a function parameter to `claimFromFactory` without the need for calling the `artData` function.

### G-04: Multiple calls to PhiReward::computeMintReward during NFT claiming

During a call to `PhiFactory::claim`, the `PhiReward::computeMintReward` function is called 4 times as shown below.
(please note: `PhiFactory::getArtMintFee` calls `PhiReward::computeMintReward`)

1. https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L300 or https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L283
1. https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L709
1. https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L738
1. Then it calls `PhiNFT1155.claimFromFactory` which in turn calls https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L134

Moreover, each call to `PhiFactory::computeMintReward`, reads four storage variables on the `PhiRewards`. This is a significant gas overhead.

#### Recommendation

Refactor to call this method only once.

### G-05: Unnecessary parameter in CuratorRewardsDistributor::deposit

Code in question: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L68

The `amount` parameter is redundant as it is asserted to be the same as the `msg.value`.

#### Recommendation

Remove the `amount` parameter and use `msg.value` instead.

### G-06: Variable packing in PhiRewards

Code in question https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L24-L27

The above storage variables take up four slots which is more than what is necessary to store these values. It is reasonable to expect they should
not be very large. They are all accessed in the `PhiRewards::computeMintReward` function so there are gas savings to be had by reducing the number of storage slots
read.

#### Recommendation

All four variables could be made into `uint128`, thus taking up two slots. Alternatively, the units of the variables could be changed from wei to microether.
Then all of them could be changed to `uint64` - thus all of them would take only one slot. The maximum value storable would be
`(2**64 - 1) * (10**(-6)) ether` = `1.8446744 * 10**13 ether` which is more than enough, and the resolution of a millionth of an ether should be sufficient.

### G-07: Unnecessary usage of bytes data type in calldata

Code in question: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L197

`bytes` type is used as a function parameter type. It is dynamically sized, implying that when abi encoded it will include 32 bytes for storing the offset where the
data begins, and also 32 bytes for the length. This is a waste because we know that the actual number of parameters contained within the `bytes` as it is abi decoded
as follows:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L208

Then the `credData` is decoded:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L557-L558

#### Recommendation

Remove the usage of the `bytes` parameter and instead list out all the underlying arguments as either individual arguments or use struct(s) to contain the individual
arguments if it makes the code more readable.

### G-08: Unused variable of PhiCred struct

Code in question: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/interfaces/ICred.sol#L133

The value of `latestActiveTimestamp` is never read anywhere in the codebas.

#### Recommendation

If this is needed off-chain, consider using events to retrieve this information instead of writing it to storage.

### G-09: Functionality of Claimable abstract contract not strictly necessary

The `Claimable` abstract contract is extended by `PhiNFT1155`. Users can call the `Claimable::signatureClaim` or `Claimable::merkleClaim` on the `PhiNFT1155`.
This results in the following call sequence:

1. EOA calls `PhiNFT1155::signatureClaim`
1. `PhiNFT1155` calls `PhiFactory::signatureClaim`
1. `PhiFactory` calls `PhiNFT1155::claimFromFactory` (i.e. calls back to the first contract called in the transaction)

Each call has non-zero `msg.value` so incurs a gas penalty of 6700.

#### Recommendation

The logic of `PhiNFT1155::signatureClaim` could be done off-chain and the first call in the sequence could be omitted.

### G-10: Storage of cred IDs per address on Cred contract

Code in question: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L49-L53

Any state update functions on the `Cred` contract that change the cred shares held by an address incur gas costs tied to updating the above storage variables. These variables are only
read via view functions. It is assumed these are intended only for consumption by client applications.

#### Recommendation

##### Option 1: Remove the storage variables and use events to determine position of an address

Emit events during each buy/sell which show the cred ID and quantity held by the address at the completion of the trade. For example:

```
event BalanceUpdate(
    address indexed curator,
    uint256 indexed credId,
    uint256 amount
)
```

On the client-side query the latest event(s) for the user's address to determine their current position.

##### Option 2: Improve the efficiency of the existing implementation

1. The `_credIdsPerAddressArrLength` storage variable can be removed as the array lengths are already stored on the `_credIdsPerAddress` variable.
1. The `_credIdExistsPerAddress` storage variable can also be removed. It is only read as part of `getPositionsForCurator` function [here](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L511). However, because the arrays within `_credIdsPerAddress` have elements added/removed appropriately already [here](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L671) and [here](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L677), then the conditional on 511 will always be `true`. This also removes the need to resize the arrays [here](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L520-L521).

### Appendix: Usage of `call`s to PhiFactory from PhiFactory

Although this issue was reported previously, this section is included to show how much gas can be saved by fixing this issue, including some basic gas usage testing.

The `PhiFactory` makes `call`s to itself in several places. For example, within the `claim` function:

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L283

This introduces an unnecessary gas overhead. Costs include:

1. `call` with non-zero value: penalty of 6700.
1. (Warm) address access cost of 100.
1. Cost associated with matching the function selector, encoding/decoding function arguments etc.

#### Recommendation

Refactor the contract to remove the usage of the `call`s from the `PhiFactory` to itself. The way that the contract accounts for the amount of
ether needed to pay for claiming needs to be changed also, because the `msg.value` is checked every time `_processClaim`
is called. If `_processClaim` is executed several times (such as during `batchClaim`), then it no longer makes sense to
check `msg.value` during each execution of `_processClaim` after the refactor (as it will be the same every time). The mint fee amount
is paid from the `PhiFactory` during the claiming process (split between `protocolFeeDestination` and the art contract). Assuming the
`protocolFeeDestination` is not set to the `PhiFactory` itself, then it is safe to simply do a before/after balance check of the `PhiFactory`
to see that sufficient `msg.value` has been sent:

```
uint256 initialBalance = address(this).balance
// Do the claiming functionality here (including fee payments) ...
// Then check how much was spent:
uint256 spent = initialBalance - address(this).balance;
if (spent > msg.value) revert InvalidMintFee();
if (spent < msg.value) {
    // Refund
    msg.sender.safeTransferETH(msg.value - spent);
}
```

An additional benefit of this refactoring is it removes the need for the second parameter of `batchClaim` - an array of values used
in the original version used to determine the `value` supplied in each `call` to `claim`.

A POC of this refactoring is available [here](https://github.com/hgrano/2024-08-phi-audit/commit/86880f3c66c7c2c56b0f7eb06e3b8c0c6daa62a7).

The test cases below show the gas usage reduction. The following command was run to execute the tests:

`forge test --match-test <Test case> -vvv --gas-report`

Then the specific amount gas used by the function was recorded using both the original version and the refactored version. Within each test case, the function in question was called only once.

| Test case | Function call | Gas Usage Reduction |
| --- | --- | --- |
| `test_claimMerkle` | `claim` |  12,031 |
| `test_claim_1155_with_ref` | `claim` | 12,669 |
| `test_batchClaim_1155_with_ref` | `batchClaim` (input containing two claims) | 42,360 |
