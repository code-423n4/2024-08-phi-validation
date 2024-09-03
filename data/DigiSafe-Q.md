### Incorrect access modifiers on `_addCredIdPerAddress` & `_removeCredIdPerAddress` allowing anyone to update private data

**Description:** 
`Cred.sol` gives users functionality to buy or sell shares. Whenever someone makes a trade he becomes a Curator and his share balance are being updated. 
The function `Cred.sol:_updateCuratorShareBalance` does this update and
it uses `Cred.sol:_addCredIdPerAddress` & `Cred.sol:_removeCredIdPerAddress` internally for managing which users own a credential. 
These two functions are intended to be internal but their modifiers are public, giving opportunity for everyone to use them and update this internal data.

**Impact:** 
Anyone can update the internal data of the `Cred.sol`

**Proof of Concept:** 
Open `Cred.sol` and scroll to both functions to see their modifiers

```javascript
// Function to add a new credId to the address's list
> function _addCredIdPerAddress(uint256 credId_, address sender_) public {
    // Add the new credId to the array
    _credIdsPerAddress[sender_].push(credId_);
    // Store the index of the new credId
    _credIdsPerAddressCredIdIndex[sender_][credId_] = _credIdsPerAddressArrLength[sender_];
    // Increment the array length counter
    _credIdsPerAddressArrLength[sender_]++;
}

// Function to remove a credId from the address's list
> function _removeCredIdPerAddress(uint256 credId_, address sender_) public {
    // Check if the array is empty
    if (_credIdsPerAddress[sender_].length == 0) revert EmptyArray();

...  
```

**Recommended Mitigation:** Change their modifiers to internal

### Missing upper bound check inside `CreatorRoyaltiesControl.sol:_updateRoyalties`

**Description:** 
Art creators can update their settings, including their royalty. The function responsible for the royalty updates are `CreatorRoyaltiesControl.sol:_updateRoyalties` and it is missing an upper bound check.
A similar issue is mentioned in the known issues, but for the mintFee

**Impact:** Art creator can set his art royalty greater than 100%

**Proof of Concept:** See the function `CreatorRoyaltiesControl.sol:_updateRoyalties`, it is missing the following check

```javascript
if (configuration.royaltyBPS > ROYALTY_BPS_TO_PERCENT) {
  revert(); //custom error
}
```

**Recommended Mitigation:** Include the upper bound check

```diff
function _updateRoyalties(uint256 tokenId, RoyaltyConfiguration memory configuration) internal {
 if (configuration.royaltyRecipient == address(0) && configuration.royaltyBPS > 0) {
 revert InvalidRoyaltyRecipient();
 }

+    if (configuration.royaltyBPS > ROYALTY_BPS_TO_PERCENT) {
+        revert(); //custom error
+    }

 royalties[tokenId] = configuration;

 emit UpdatedRoyalties(tokenId, msg.sender, configuration);
 }
```

### Upper bound check prevents updating the fees in `PhiFactory.sol`

**Description:** 
Minting and art creation have fees for the protocol. 
They are represented as `artCreateFee` and `mintProtocolFee` which are updated using `PhiFactory.sol:setProtocolFee` and `PhiFactory.sol:setArtCreatFee`.
The problem is that these two setters expect the input to be a percentage in BPS and have an upper bound check for the BPS, but the variables are not percentages and this check prevents the updating of these variables with a reasonable value

**Impact:** 
Inability to update the `mintProtocolFee` and `artCreateFee` with a reasonable value

**Proof of Concept:** 
Open `PhiFactory.sol` and see the conditionals inside

**Recommended Mitigation:** 
Replace the upper bound check with a greater fair value

```diff
/// @notice Sets the protocol fee percentage.
/// @param protocolFee_ The new protocol fee percentage (in basis points).
function setProtocolFee(uint256 protocolFee_) external onlyOwner {
- if (protocolFee_ > 10_000) revert ProtocolFeeTooHigh();
+ if (protocolFee_ > SOME_OTHER_FAIR_CONSTANT) revert ProtocolFeeTooHigh();
 mintProtocolFee = protocolFee_;
 emit ProtocolFeeSet(protocolFee_);
}

/// @notice Sets the art creation fee percentage.
/// @param artCreateFee_ The new art creation fee percentage (in basis points).
function setArtCreatFee(uint256 artCreateFee_) external onlyOwner {
-  if (artCreateFee_ > 10_000) revert ArtCreatFeeTooHigh();
+ if (artCreateFee_ > SOME_OTHER_FAIR_CONSTANT) revert ArtCreatFeeTooHigh();
 artCreateFee = artCreateFee_;
 emit ArtCreatFeeSet(artCreateFee_);
}
```

### Missing gap in upgradeable `CreatorRoyaltiesControl.sol`

**Description:** 
Note, the same issue has been mentioned in the automated findings, but for another instance (L-18)
`PhiNFT1155.sol` is meant to be upgradeable. 
However, it inherits `CreatorRoyaltiesControl.sol`, which is not upgrade-safe, because it does not implement a `gap` storage. 
Adding a new variable to `CreatorRoyalitesControl.sol` will overwrite the beginning of the storage layout of the child contracts
(`PhiNFT1155.sol`)

**Impact:** `PhiNFT1155.sol`'s can be corrupted after an upgrade, causing critical misbehaviour in the system.

**Proof of Concept:** 
An example will be the `PhiNFT1155.sol` which inherits the `CreatorRoyalitesControl.sol` contract.
If a new storage variable is added to the `CreatorRoyalitesControl.sol` contract, it will overwrite the `phiFactoryContract` variable in the `PhiNFT1155.sol`, causing unintended consequences.

**Recommended Mitigation:** 
Add a storage gap in `CreatorRoyalitesControl.sol`:

```diff
abstract contract CreatorRoyaltiesControl is ICreatorRoyaltiesControl {
 mapping(uint256 _tokenId => RoyaltyConfiguration _configuration) public royalties;
 uint256 private constant ROYALTY_BPS_TO_PERCENT = 10_000;
 address private royaltyRecipient;
 bool private initilaized;

+   uint256[50] private __gap;

 ...
```