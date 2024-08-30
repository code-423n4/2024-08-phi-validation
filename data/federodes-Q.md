[L01] `PhiNFT1155::createArtFromFactory` reverts when `msg.value > artFee`.

The issue comes from the fact that the caller of the `createArtFromFactory` is the `PhiFactory` contract (notice the `onlyPhiFactory` modifier), and said contract cannot receive ETH via transfer calls due to the fact that it does have a `receive` or `fallback` call implemented.

```
    function createArtFromFactory(uint256 artId_) external payable onlyPhiFactory whenNotPaused returns (uint256) {
        _artIdToTokenId[artId_] = tokenIdCounter;
        _tokenIdToArtId[tokenIdCounter] = artId_;

        uint256 artFee = phiFactoryContract.artCreateFee();

        protocolFeeDestination.safeTransferETH(artFee);
        emit ArtCreated(artId_, tokenIdCounter);
        uint256 createdTokenId = tokenIdCounter;

        unchecked {
            tokenIdCounter += 1;
        }
        if ((msg.value - artFee) > 0) {
            //@> The following call will always revert due to the PhiFactory contract is not able to receive ETH via regular transactions.
            _msgSender().safeTransferETH(msg.value - artFee);
        }

        return createdTokenId;
    }
```

Fix:

Either implement a `receive` or `fallback` functions in the `PhiFactory` contract or require that `msg.value == artFee` inside `createArtFromFactory`.

//=======================================

[L02] `PhiFactory::createArt` reverts when sending more than NFT_ART_CREATE_FEE as msg.value.

This issue is related to [L01] above, where the internal call made from `PhiFactory::createERC1155Internal` to `PhiNFT1155::createArtFromFactory` reverts due to the `PhiFactory` contract is unable to receive ETH via transfer.

Now, even if `PhiFactory::createERC1155Internal` would be able to receive ETH via transfer, the current implementation of `PhiFactory::createArt` would not refund users that send more than the artFee as msg.value since the logic to reimbursing users is not implemented.

```
function createArt(
        bytes calldata signedData_,
        bytes calldata signature_,
        CreateConfig memory createConfig_
    )
        external
        payable
        nonReentrant
        whenNotPaused
        returns (address)
    {
        _validateArtCreationSignature(signedData_, signature_);
        (, string memory uri_, bytes memory credData) = abi.decode(signedData_, (uint256, string, bytes));
        ERC1155Data memory erc1155Data = _createERC1155Data(artIdCounter, createConfig_, uri_, credData);
        address artAddress = createERC1155Internal(artIdCounter, erc1155Data);
        artIdCounter++;
        return artAddress;
    }`
```

//=======================================
