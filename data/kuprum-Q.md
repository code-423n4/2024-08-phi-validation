## [L-1] Arts can be created with starting time in the past

### Links to affected code

- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L585-L586

### Impact

Arts can be created with the starting time in the past, thus opening the minting window earlier than intended.

[_validateArtCreation](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L585-L586) performs only the following checks:
 
```solidity
if (createData_.endTime <= block.timestamp) revert EndTimeInPast();
if (createData_.endTime <= createData_.startTime) revert EndTimeLessThanOrEqualToStartTime();
```

These checks still allow to have `startTime < block.timestamp`.

## [L-2] Cross-chain signature replay possible in Cred::updateCred

### Links to affected code

- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L283-L310

### Impact

Function [Cred::updateCred](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L283-L310) doesn't validate the chain id:

```solidity
if (_recoverSigner(keccak256(signedData_), signature_) != phiSignerAddress) revert AddressNotSigned();
(uint256 expiresIn_, address sender,, uint256 credId, string memory credURL) =
    abi.decode(signedData_, (uint256, address, uint256, uint256, string));
```

`chainId` is encoded between `sender` and `credId`, but is ignored. This can be employed to replay the signature obtained for one chain on another chain. While, due to other checks present in the function, no immediate negative consequences seem to exist, this is still a serious vulnerability worth fixing.

### Recommended Mitigation Steps

Validate that `chainId` matches `block.chainid`.



## [L-3] Incorrect array indices used in `Cred::getPositionsForCurator`

### Links to affected code

- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L513-L514

### Impact

[Cred::getPositionsForCurator](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L480-L523) uses incorrect indices for array updates:

```solidity
uint256 index = 0;
for (uint256 i = start_; i < stopIndex; i++) {
    uint256 credId = userCredIds[i];
    if (_credIdExistsPerAddress[curator_][credId]) {
        uint256 amount = shareBalance[credId].get(curator_);
        credIds[i] = credId;   // <<== wrong; should be `index` instead of `i`
        amounts[i] = amount;   // <<== wrong; should be `index` instead of `i`
        index++;
    }
}
// Resize the result array to remove unused slots
assembly {
    mstore(credIds, index)
    mstore(amounts, index)
}
```

If there were gaps in `userCredIds` (i.e. elements for which `_credIdExistsPerAddress[curator_][credId]` evaluates to `false`), then `credIds` and `amounts` would be updated wrongly, and then the resulted arrays would be wrongly truncated by the assembly fragment. To the best of our knowledge, this situation is not possible with the current code base; but the code is nevertheless incorrect.

### Recommended Mitigation Steps

Replace the two wrong lines above with 

```solidity
        credIds[index] = credId;
        amounts[index] = amount;
```

## [L-4] `Cred` contract uses `uint40` to hold `block.timestamp`, which is too narrow

### Links to affected code

- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/interfaces/ICred.sol#L131-L132
- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L305
- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L566

### Impact

The `Cred` contract employs `uint40` to hold timestamps in the `createdAt` and `updatedAt` fields of the [PhiCred struct](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/interfaces/ICred.sol#L131-L132): the datatype wraps around even now. While no immediate negative consequences can be observed in the present codebase, this may still pose problems for systems integrating with Phi.


### Recommended Mitigation Steps

Employ at least `uint64` to hold timestamps.


## [L-5] `Cred::_validateAndCalculateBatch` doesn't validate the length of `priceLimits_` input array

### Links to affected code

- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L810-L828

### Impact

[`Cred::_validateAndCalculateBatch`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L810-L828) has three input arrays, but only two are validated to have equal length, namely `credIds_` and `amounts_`; length of `priceLimits_` is not validated to be the same. This may lead to unexpected behaviors, or unexpected reverts with "Array out of bounds" error message.

### Recommended Mitigation Steps

Validate that all three input arrays have equal lengths.

## [L-6] Sender validation in `PhiFactory::_validateAndUpdateClaimState` allows calling `signatureClaim` and `merkleClaim` from any EOA address

### Links to affected code

- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L705-L708

### Impact

[PhiFactory::_validateAndUpdateClaimState](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L702C14-L721), which is called from `signatureClaim` and `merkleClaim`, validates the sender as follows:

```solidity
// Common validations
if (tx.origin != _msgSender() && msg.sender != art.artAddress && msg.sender != address(this)) {
    revert TxOriginMismatch();
}
```

The apparent intention is for the checks to allow calling functions `signatureClaim` and `merkleClaim` to be callable only from the `PhiNFT1155` contract. Unfortunately, the checks also allow to call these functions directly from any EOA address, for which `tx.origin == _msgSender()`.

## [L-7] `RewardControl::withdrawFor` allows anyone to move others' funds

### Links to affected code

- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L96-L98

### Impact

Function [RewardControl::withdrawFor](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L96-L98) allows anyone, unrestricted, to withdraw funds from `RewardControl` to that person associated account. While this might not pose immediate danger, still allowing unrestricted movement of funds for others seems questionable, as this may have unforeseeable consequences; e.g. tax-related implications.

### Recommended Mitigation Steps

Remove function `RewardControl::withdrawFor`.

## [L-8] `PhiNFT1155` is not ERC-1155 compliant: `createArtFromFactory` doesn't emit the `TransferSingle` event.

### Links to affected code

- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L138-L156

### Impact

According to [ERC-1155](https://eips.ethereum.org/EIPS/eip-1155):

> - A mint/create operation is essentially a specialized transfer and MUST follow these rules:
>    - To broadcast the existence of a token ID with no initial balance, the contract SHOULD emit the `TransferSingle` event from `0x0` to `0x0`, with the token creator as `_operator`, and a `_value` of 0.

The creation function of the `PhiNFT1155` contract, [createArtFromFactory](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L138-L156), doesn't follow this rule, i.e. it doesn't emit the `TransferSingle` event as required, thus making `PhiNFT1155` not ERC-1155 compliant.

## [L-9] `PhiFactory::merkleClaim` can be called anytime by anyone, any number of times

### Links to affected code

- https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L352-L383

### Impact

[PhiFactory::merkleClaim](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L352-L383) doesn't validate the sender, allowing anyone to replay the signature.

Also, unlike [PhiFactory::signatureClaim](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L327-L346) for which the deadline is included into the signed data and checked against `block.timestamp`, no such validation is done for [PhiFactory::merkleClaim](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L352-L383).

Combined, the above leads to the ability for anyone to reuse the same signature in order to mint unbounded number of NFTs (up to `maxSupply`) over unbounded time horizon.

After consulting with the sponsor, it has been confirmed though that unlimited minting of NFTs via merkleClaim (anytime, any amount) is the intended behavior; we thus refrained of submitting this finding independently as Medium or High; we are submitting it here as Low instead. We still consider it reasonable to impose additional restrictions on who and when is allowed to call the function.

### Proof of Concept

Drop this test to [Claimable.t.sol(https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/test/Claimable.t.sol#L172) and execute via `forge test --match-test Kuprum`

```solidity
function testKuprum_MultipleMerkleClaim() public {
    uint256 artId = 2;

    bytes32[] memory proof = new bytes32[](2);
    proof[0] = 0x0927f012522ebd33191e00fe62c11db25288016345e12e6b63709bb618d777d4;
    proof[1] = 0xdd05ddd79adc5569806124d3c5d8151b75bc81032a0ea21d4cd74fd964947bf5;

    address minter = 0x1111111111111111111111111111111111111111;
    address ref = referrer;
    uint256 tokenId = 2;
    uint256 quantity = 1;
    bytes32 leafPart = 0x0000000000000000000000000000000003c2f7086aed236c807a1b5000000000;
    string memory imageURI = ART_ID2_URL_STRING;

    bytes memory claimData = abi.encode(minter, ref, artId);
    IPhiFactory.MintArgs memory mintArgs = IPhiFactory.MintArgs(tokenId, quantity, imageURI);

    // Claim the first time
    vm.warp(START_TIME + 1);
    vm.deal(user1, phiFactory.getArtMintFee(artId, 1));
    vm.startPrank(user1, user1);
    phiFactory.merkleClaim{ value: phiFactory.getArtMintFee(artId, 1) }(proof, claimData, mintArgs, leafPart);

    // Claim the second time with the same parameters, but from a different EOA
    vm.warp(START_TIME + 12 hours);
    vm.deal(referrer, phiFactory.getArtMintFee(artId, 1));
    vm.startPrank(referrer, referrer);
    phiFactory.merkleClaim{ value: phiFactory.getArtMintFee(artId, 1) }(proof, claimData, mintArgs, leafPart);
}
```

### Recommended Mitigation Steps

- Add the deadline to the Merkle proof tree, and verify it in `PhiFactory::merkleClaim` similar to `PhiFactory::signatureClaim`.
- Validate that `PhiFactory::merkleClaim` is called by the `minter_`.
