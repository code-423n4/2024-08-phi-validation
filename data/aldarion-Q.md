### 1. Excess payment sent to PhiFactory when art is created
Context:
 https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L152

Detail:
`PhiNFT1155.createArtFromFactory()` refund any excess ETH to caller but the caller for this function is `PhiFactory`.This results in any excess ETH being refunded to PhiFactory rather than the user. User lost this funds.

Recommendation:
 Use different logic to refund or enforce that the amount of ETH sent with the call matches the required fee exactly. This prevents any excess ETH from needing to be refunded.

### 2. Excess payment sent to PhiFactory when using factory claim

Context:
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L283

Detail:
The issue arises due to the `PhiFactory.claim()` function making external calls to itself (`this.merkleClaim` and `this.signatureClaim`). When it uses this to call its functions, the `msg.sender` within those function calls becomes the factory contract itself instead of the original external caller (the user).

```solidity
...
this.merkleClaim{ value: mintFee }(proof, claimData, mintArgs, leafPart_);
...
this.signatureClaim{ value: mintFee }(signature_, claimData, mintArgs);
```
When these functions try to refund any excess ETH to the caller, they are inadvertently sending the refund back to the `PhiFactory` contract itself, not the original user. This means the user loses the excess ETH they were supposed to receive back.

```solidity
    function _processClaim(){
...
        // Handle refund
        uint256 mintFee = getArtMintFee(artId_, quantity_);
        if ((etherValue_ - mintFee) > 0) {
            _msgSender().safeTransferETH(etherValue_ - mintFee);
        }
}
```
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L740

### Test:
```solidity
    function test_ExcessETH() public {
        _createArt(ART_ID_URL_STRING);
        uint256 artId = 1;
        bytes32 advanced_data = bytes32("1");
        bytes memory signData =
            abi.encode(expiresIn, participant, referrer, verifier, artId, block.chainid, advanced_data);
        bytes32 msgHash = keccak256(signData);
        bytes32 digest = ECDSA.toEthSignedMessageHash(msgHash);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(claimSignerPrivateKey, digest);
        if (v != 27) s = s | bytes32(uint256(1) << 255);
        bytes memory signature = abi.encodePacked(r, s);
        bytes memory data =
            abi.encode(1, participant, referrer, verifier, expiresIn, uint256(1), advanced_data, IMAGE_URL, signature);
        bytes memory dataCompressed = LibZip.cdCompress(data);
        uint256 totalMintFee = phiFactory.getArtMintFee(1, 1);

        vm.startPrank(participant, participant);
        assertEq(address(phiFactory).balance, 0);

        phiFactory.claim{ value: 2*totalMintFee }(dataCompressed);
        assertEq(address(phiFactory).balance, totalMintFee);
```

Recommendation:
 Use different logic to refund or enforce that the amount of ETH sent with the call matches the required fee exactly. This prevents any excess ETH from needing to be refunded.