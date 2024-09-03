# Quality Assurance Report

## Table of Contents
| Issue ID | Description |
|----------|-------------|
| [QA-01](#qa-01-contracturi-not-return-generic-info) | contractURI not return generic info |
| [QA-02](#qa-02-artcreatefee-set-in-phifactory-as-percentage-but-used-as-whole-number) | artCreateFee set in PhiFactory as percentage but used as whole number |
| [QA-03](#qa-03-mintprotocolfee-defined-as-percentage-but-used-as-whole-value) | mintProtocolFee defined as percentage but used as whole value |
| [QA-04](#qa-04-missing-check-for-array-length-mismatch-between-artid--quantitys-) | missing check for array length mismatch between artId_ & quantitys_ |
| [QA-05](#qa-05-phifactorycreateart-allows-input-signature-to-be-replayed-due-to-lacking-nonce) | PhiFactory:createArt allows input signature to be replayed due to lacking nonce |
| [QA-06](#qa-06-phifactorycreateart-reverts-if-artcreatefee-drop) | PhiFactory:createArt reverts if `artCreateFee` drop |
| [QA-07](#qa-07-import-test-library-to-production-code) | import test library to production code |

## [QA-01] contractURI not return generic info
### Impact
The `contractURI` method is only valid to build description for `tokenId 1` as Art with different `credCreators` and `verificationType` can co-exist in the same `PhiNFT1155` contract with different `artIds`, leading to potential inconsistencies.

### Instances
Returned JSON contains artId dependent info via `_buildDescription(art)` where properties like `credId`, `credChainId`, `credCreator`, and `verificationType` were used to construct the contract description. However, aside from `credId` and `credChainId`, the `credCreator` and `verificationType` are `artId` specific, which could be recorded to `tokenId` different from 1 via `_useExistingNFTContract`.
  
  Found in `src/PhiFactory.sol` at [Line 65](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L65)
  
  ```solidity
  64:    function contractURI(address nftAddress) public view returns (string memory) {
  65: =>         uint256 artId = IPhiNFT1155Ownable(nftAddress).getFactoryArtId(1); 
  66:        PhiArt memory art = arts[artId];
  ...
  92:    }
  ```

The function `contractURI` uses `_buildDescription(art)` to build the description, which may contain art-specific information rather than generic data.
  
  Found in `src/PhiFactory.sol` at [Line 75](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L75)
  
  ```solidity
  64:    function contractURI(address nftAddress) public view returns (string memory) {
          ...
  74:                '"description":"',
  75: =>                 _buildDescription(art),
  76:                '",',
                  ...
  92:    }
  ```

Art with different `credCreators` and `verificationType` can co-exist in the same `PhiNFT1155` contract with different `artIds`, leading to potential inconsistencies.
  
  Found in `src/PhiFactory.sol` at [Line 772](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L772)
  
  ```solidity
  763:    function _buildDescription(PhiArt memory art) internal pure returns (string memory) {
          ...
  771:                "This NFT represents a unique on-chain cred created by ",
  772: =>                 art.credCreator.toHexString(),
  773:                ". ",
  774:                "The cred is verified using ",
  775: =>               art.verificationType,
  776:                ". ",
                  ...
  781:    }
  ```

### Recommendation
Make the NFT description more generic to ensure it accurately represents all `tokenId`s and not just `tokenId 1`.

---

## [QA-02] artCreateFee set in PhiFactory as percentage but used as whole number
### Impact
If `artCreateFee` was intended as a percentage as documented in the code, the usage in `createArtFromFactory` would be incorrect, potentially breaking the core functionality of the protocol.

### Instances
The art creation fee is defined as a percentage in basis points, but the documentation and usage imply it is used as a whole number.
  
  Found in `src/PhiFactory.sol` at [Line 126](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L126)
  
  ```solidity
  126: =>     /// @param artCreateFee_ The art creation fee percentage (in basis points). 
  ```

The art creation fee percentage should be in basis points, denominated by 10_000, but it is being used directly as a whole number ie. `artCreateFee`'s return value was passed to `safeTransferETH` instead of deducing a percentage of specific amount.
  
  Found in `src/art/PhiNFT1155.sol` at [Line 142](https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L142)
  
  ```solidity
  138:    function createArtFromFactory(uint256 artId_) external payable onlyPhiFactory whenNotPaused returns (uint256) {
          ...
  141:
  142: =>         uint256 artFee = phiFactoryContract.artCreateFee(); 
  143:
  144: =>       protocolFeeDestination.safeTransferETH(artFee);
          ...
  156:    }
  ```

### Recommendation
Update the developer's documentation for `PhiFactory:initialize`'s `artCreateFee_` parameter and `setArtCreateFee` to clarify the intended use of `artCreateFee` as a percentage in basis points.

---

## [QA-03] mintProtocolFee defined as percentage but used as whole value
### Impact
The `mintProtocolFee` is intended to be used as a percentage (in basis points), but it is used as a whole number, which can result in incorrect fee calculations and potential financial losses.

### Instances
The protocol fee is set as a percentage but used as a whole number.
  
  Found in `src/PhiFactory.sol` at [Line 421](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L421)
  
  ```solidity
  421: =>     /// @param protocolFee_ The new protocol fee percentage (in basis points). 
  ```

The `mintProtocolFee` is defined as a percentage but used as a whole value.
  
  Found in `src/PhiFactory.sol` at [Line 511](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L511)
  
  ```solidity
  509:    function getArtMintFee(uint256 artId_, uint256 quantity_) public view returns (uint256) {
  510:        return IPhiRewards(phiRewardsAddress).computeMintReward(quantity_, arts[artId_].mintFee)
  511: =>             + quantity_ * mintProtocolFee; 
  512:    }
  ```

---

## [QA-04] missing check for array length mismatch between artId_ & quantitys_
### Impact
If there is an array length mismatch between `artId_` and `quantitys_`, the function will consume unnecessary gas before reverting, leading to higher transaction costs.

### Instances
Missing check for array length mismatch between `artId_` and `quantitys_`.
  
  Found in `src/PhiFactory.sol` at [Line 528](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L528)
  
  ```solidity
  517:    function getTotalMintFee(
          ...
  527:            uint256 artId = artId_[i];
  528: =>             totalMintFee = totalMintFee + getArtMintFee(artId, quantitys_[i]); 
  529:        }
          ...
  531:    }
  ```

### Recommendation
```solidity
if (artId_.length != quantitys_.length) revert ArrayLengthMismatch();
```

---

## [QA-05] PhiFactory:createArt allows input signature to be replayed due to lacking nonce
### Impact
Without nonce validation, a `createArt` transaction can be replayed, leading to potential double spending or unauthorized actions.

### Instances
Signature derived from `signedData` which contains only `expires` property and is missing `nonce` which is needed to prevent replay attacks.
  
  Found in `src/PhiFactory.sol` at [Line 589](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L589)
  
  ```solidity
  589: =>     function _validateArtCreationSignature(bytes calldata signedData_, bytes calldata signature_) private view {
  590:        if (_recoverSigner(keccak256(signedData_), signature_) != phiSignerAddress) revert AddressNotSigned();
  591:        (uint256 expiresIn_,,) = abi.decode(signedData_, (uint256, string, bytes));
  592:        if (expiresIn_ <= block.timestamp) revert SignatureExpired();
  593:    }
  ```

### Recommendation
Include a nonce in the `signedData` struct to prevent replay attacks.

---

## [QA-06] PhiFactory:createArt reverts if `artCreateFee` drop
### Impact
The `PhiFactory` contract cannot receive refund ETH as it does not implement a `payable` fallback function. If `msg.value > artFee` in PhiNFT1155:createArtFromFactory call, the transaction will revert.

### Instances
`createArtFromFactory` expects to refund excess ETH to `msgSender`, which is `PhiFactory` contract but not possible due to missing payable fallback in `PhiFactory` code.
  
  Found in `src/art/PhiNFT1155.sol` at [Line 152](https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L152)
  
  ```solidity
  138:    function createArtFromFactory(uint256 artId_) external payable onlyPhiFactory whenNotPaused returns (uint256) {
          ...
  151:        if ((msg.value - artFee) > 0) {
  152: =>             _msgSender().safeTransferETH(msg.value - artFee); 
  153:        }
  ...
  156:    }
  ```
### POC
> Apply this patch & Run with: forge test -vvv --mt test_createArt

A PASS result confirms `createArt` will revert if extra ETH was sent.

```patch
diff --git a/test/PhiFactory.t.sol b/test/PhiFactory.t.sol
index f999052..b4e528c 100644
--- a/test/PhiFactory.t.sol
+++ b/test/PhiFactory.t.sol
@@ -70,6 +70,21 @@ contract TestPhiFactory is Settings {
         phiFactory.createArt{ value: NFT_ART_CREATE_FEE }(signCreateData, abi.encodePacked(cr, cs), config);
     }
 
+    function test_createArt() public {
+        bytes memory credData = abi.encode(1, owner, "SIGNATURE", 31_337, bytes32(0));
+        bytes memory signCreateData = abi.encode(expiresIn, "sample-art-id", credData);
+        bytes32 createMsgHash = keccak256(signCreateData);
+        bytes32 createDigest = ECDSA.toEthSignedMessageHash(createMsgHash);
+        (uint8 cv, bytes32 cr, bytes32 cs) = vm.sign(claimSignerPrivateKey, createDigest);
+        if (cv != 27) cs = cs | bytes32(uint256(1) << 255);
+        IPhiFactory.CreateConfig memory config =
+            IPhiFactory.CreateConfig(artCreator, receiver, END_TIME, START_TIME, MAX_SUPPLY, MINT_FEE, false);
+
+        // 3. refund reverts
+        vm.expectRevert();
+        phiFactory.createArt{ value: NFT_ART_CREATE_FEE + 1 }(signCreateData, abi.encodePacked(cr, cs), config); // send extra 1 wei
+    }
+
     function test_constructor() public view {
         assertEq(phiFactory.owner(), owner, "owner is correct");
         assertEq(phiFactory.protocolFeeDestination(), protocolFeeDestination, "protocolFeeDestination is correct");

```

### Recommendation
Implement a payable fallback function in the `PhiFactory` contract to handle ETH refunds.

---

## [QA-07] import test library to production code
### Impact
Including test libraries in production code leads to larger bytecode and increased deployment costs.

### Instances
Importing the test library `console2` from `forge-std` in production code.
  
  Found in `src/curve/BondingCurve.sol` at [Line 9](https://github.com/code-423n4/2024-08-phi/blob/main/src/curve/BondingCurve.sol#L9)
  
  ```solidity
  9: => import { console2 } from "forge-std/console2.sol";
  ```

Another instance of importing the test library `console2` from `forge-std` in production code.
  
  Found in `src/reward/CuratorRewardsDistributor.sol` at [Line 12](https://github.com/code-423n4/2024-08-phi/blob/main/src/reward/CuratorRewardsDistributor.sol#L12)
  
  ```solidity
  12: => import { console2 } from "forge-std/console2.sol"; 
  ```

### Recommendation
Remove the import of the test library from the production code.