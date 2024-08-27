| Issue ID | Issue Title |
|----------|-------------|
| [L-01](#l-01-potential-for-signature-replay-attack-in-withdrawwithsig-function) | Potential for signature replay attack in `withdrawWithSig` function |
| [L-02](#l-02-inconsistent-token-uri-handling-in-phinft1155-contract) | Inconsistent Token URI Handling in `PhiNFT1155` Contract |
| [L-03](#l-03-phinft1155-is-not-erc1155-compliant) | PhiNFT1155 is not ERC1155 compliant |
| [L-04](#l-04-lack-of-proper-initialization-and-validation-for-credcontract-can-lead-to-unexpected-behavior) | Lack of proper initialization and validation for `credContract` can lead to unexpected behavior |
| [L-05](#l-05-potential-precision-loss-in-reward-calculations) | Potential Precision Loss in Reward Calculations |


## [L-01] Potential for signature replay attack in `withdrawWithSig` function

### Vulnerability Detail

The `withdrawWithSig` function in the `RewardControl` contract is designed to allow users to withdraw funds using a signed message. This function relies on the `_verifySignature` function to validate the signature. The `_verifySignature` function constructs a hash using the `WITHDRAW_TYPEHASH`, `from`, `to`, `amount`, `nonce`, and `deadline` parameters. However, it does not include the chain ID or the contract address in the hash.

The relevant code is as follows:

```solidity
function withdrawWithSig(address from, address to, uint256 amount, uint256 deadline, bytes calldata sig) external {
    if (block.timestamp > deadline) revert DeadlineExpired();
    if (!_verifySignature(from, to, amount, nonces[from], deadline, sig)) revert InvalidSignature();

    unchecked {
        ++nonces[from];
    }

    _withdraw(from, to, amount);
}

function _verifySignature(
    address from,
    address to,
    uint256 amount,
    uint256 nonce,
    uint256 deadline,
    bytes calldata sig
)
    internal
    view
    returns (bool)
{
    bytes32 structHash = keccak256(abi.encode(WITHDRAW_TYPEHASH, from, to, amount, nonce, deadline));
    bytes32 digest = _hashTypedData(structHash);
    return SignatureCheckerLib.isValidSignatureNowCalldata(from, digest, sig);
}
```

This omission allows a valid signature on one chain or contract instance to be replayed on another chain or instance where the nonce is still valid. For example, if a user signs a withdrawal transaction on Chain A, an attacker could replay this signature on Chain B, where the contract is also deployed and the nonce is still valid, leading to unauthorized withdrawals.

### Impact

Potential for unauthorized withdrawals across different chains or contract deployments.

### Recommendation

To prevent this vulnerability, include the chain ID and contract address in the signature hash. Update the `_verifySignature` function and the `WITHDRAW_TYPEHASH` constant as follows:

```diff
// ... existing code ...
function _verifySignature(
    address from,
    address to,
    uint256 amount,
    uint256 nonce,
    uint256 deadline,
    bytes calldata sig
)
    internal
    view
    returns (bool)
{
-    bytes32 structHash = keccak256(abi.encode(WITHDRAW_TYPEHASH, from, to, amount, nonce, deadline));
+    bytes32 structHash = keccak256(abi.encode(
+        WITHDRAW_TYPEHASH,
+        from,
+        to,
+        amount,
+        nonce,
+        deadline,
+        block.chainid,
+        address(this)
+    ));
    bytes32 digest = _hashTypedData(structHash);
    return SignatureCheckerLib.isValidSignatureNowCalldata(from, digest, sig);
}

- bytes32 public constant WITHDRAW_TYPEHASH =
-    keccak256("Withdraw(address from,address to,uint256 amount,uint256 nonce,uint256 deadline)");
+ bytes32 public constant WITHDRAW_TYPEHASH =
+    keccak256("Withdraw(address from,address to,uint256 amount,uint256 nonce,uint256 deadline,uint256 chainId,address verifyingContract)");
// ... existing code ...
```





## [L-02] Inconsistent Token URI Handling in `PhiNFT1155` Contract

### Vulnerability Detail

The `PhiNFT1155` contract has two `uri` functions with different signatures, leading to inconsistent and potentially exploitable logic for handling token URIs. The standard `uri(uint256)` function always returns the URI from the factory contract, while the custom `uri(uint256, address)` function can return a different URI based on the minter's address.

The standard `uri(uint256)` function:
```solidity
function uri(uint256 tokenId_) public view override returns (string memory) {
    return phiFactoryContract.getTokenURI(_tokenIdToArtId[tokenId_]);
}
```

The custom `uri(uint256, address)` function:
```solidity
function uri(uint256 tokenId_, address minter_) public view returns (string memory) {
    if (bytes(advancedTokenURI[tokenId_][minter_]).length > 0) {
        return advancedTokenURI[tokenId_][minter_];
    } else {
        return phiFactoryContract.getTokenURI(_tokenIdToArtId[tokenId_]);
    }
}
```

This discrepancy can lead to metadata inconsistency, where the same token can have different metadata depending on which `uri` function is called. This violates the ERC1155 standard, which specifies a single `uri(uint256)` function for retrieving token metadata. Additionally, the reliance on the factory contract for the standard `uri` function introduces a centralization risk.

### Impact

The inconsistent handling of token URIs can lead to metadata manipulation, user confusion, and potential exploitation.

### Recommendation

Standardize the URI handling to ensure consistency across all queries. If minter-specific URIs are necessary, implement a clear and transparent way to handle them that doesn't violate the ERC1155 standard. Consider using a single source of truth for token URIs, possibly by extending the standard `uri` function to include minter-specific logic. Implement proper access controls for setting and updating URIs to prevent unauthorized metadata changes.




## [L-03] PhiNFT1155 is not ERC1155 compliant

### Vulnerability Detail

The `PhiNFT1155` contract is intended to implement the ERC-1155 Multi Token Standard, which allows for the management of multiple token types within a single contract. However, the contract currently fails to comply with several critical requirements of the ERC-1155 standard, which could lead to significant interoperability and functionality issues.

1. **Missing `balanceOfBatch` Function:**
   The ERC-1155 standard requires the implementation of the `balanceOfBatch` function to allow querying the balances of multiple token IDs for multiple accounts in a single call. This function is not present in the `PhiNFT1155` contract.

   ```solidity
   function balanceOfBatch(address[] calldata _owners, uint256[] calldata _ids) external view returns (uint256[] memory);
   ```

2. **Incorrect Implementation of `uri` Function:**
   The `uri` function in the `PhiNFT1155` contract does not handle unminted token IDs correctly, potentially returning an empty or invalid URI. The ERC-1155 standard requires that the `uri` function returns a valid URI for all token types, including those that are not yet minted.

   ```solidity

   function uri(uint256 tokenId_) public view override returns (string memory) {
       return phiFactoryContract.getTokenURI(_tokenIdToArtId[tokenId_]);
   }
   ```

3. **Non-Standard Additional `uri` Function:**
   The contract includes an additional `uri` function with a different signature, which is not part of the ERC-1155 standard. This could cause compatibility issues with tools expecting the standard ERC-1155 interface.

   ```solidity

   function uri(uint256 tokenId_, address minter_) public view returns (string memory) {
       if (bytes(advancedTokenURI[tokenId_][minter_]).length > 0) {
           return advancedTokenURI[tokenId_][minter_];
       } else {
           return phiFactoryContract.getTokenURI(_tokenIdToArtId[tokenId_]);
       }
   }
   ```

4. **Overridden Transfer Functions:**
   The `safeTransferFrom` function includes a `soulBounded` check that might unexpectedly prevent transfers, potentially violating the expected behavior of ERC-1155.

   ```solidity

   function safeTransferFrom(
       address from_,
       address to_,
       uint256 id_,
       uint256 value_,
       bytes memory data_
   ) public override {
       if (from_ != address(0) && soulBounded(id_)) revert TokenNotTransferable();
       // ...
   }
   ```

5. **Incomplete Event Emissions:**
   The contract does not emit all required ERC-1155 events, particularly `TransferSingle` and `TransferBatch`, which are essential for tracking token transfers and ensuring interoperability with other contracts and tools.

### Impact

The non-compliance with the ERC-1155 standard will impact the interoperability and functionality of the `PhiNFT1155` contract. Tools and contracts that interact with ERC-1155 tokens may not function correctly with this contract, leading to potential issues in token transfers, balance queries, and URI retrievals. This could undermine the intended functionality of the contract and lead to significant operational issues.

### Recommendation

1. **Implement `balanceOfBatch` Function:**
   ```solidity

   function balanceOfBatch(address[] memory accounts, uint256[] memory ids) public view override returns (uint256[] memory) {
       require(accounts.length == ids.length, "ERC1155: accounts and ids length mismatch");
       uint256[] memory batchBalances = new uint256[](accounts.length);
       for (uint256 i = 0; i < accounts.length; ++i) {
           batchBalances[i] = balanceOf(accounts[i], ids[i]);
       }
       return batchBalances;
   }
   ```

2. **Revise `uri` Function:**
   ```solidity

   function uri(uint256 tokenId_) public view override returns (string memory) {
       if (_tokenIdToArtId[tokenId_] == 0) {
           return defaultURI; // Return a default URI for unminted tokens
       }
       return phiFactoryContract.getTokenURI(_tokenIdToArtId[tokenId_]);
   }
   ```

3. **Document Non-Standard `uri` Function:**
   Ensure the additional `uri` function is well-documented, and its purpose is clear to avoid compatibility issues.

4. **Review Transfer Restrictions:**
   Ensure the `soulBounded` check does not break ERC-1155 compatibility and is clearly documented.

5. **Emit Required Events:**
   Ensure all required ERC-1155 events (`TransferSingle` and `TransferBatch`) are emitted during transfers and approvals to maintain interoperability and correct tracking of token transfers.

For more details on the ERC-1155 standard, refer to the [ERC-1155: Multi Token Standard](https://eips.ethereum.org/EIPS/eip-1155).




## [L-04] Lack of proper initialization and validation for `credContract` can lead to unexpected behavior

### Vulnerability Detail 

The `BondingCurve` contract relies on the `credContract` for various critical functions, such as `getPriceData()`, `getBuyPriceAfterFee()`, `getSellPriceAfterFee()`, and `_getCreatorFee()`. However, the `credContract` is not initialized in the constructor, and there are no checks to ensure it is set before these functions are called. This can lead to the contract attempting to call methods on a zero address, causing reverts.

Additionally, the `setCredContract()` function can be called multiple times, allowing the owner to change the `credContract` address at will. This could lead to inconsistent behavior and potential exploitation.

The relevant code snippets are:

```solidity
ICred public credContract;

constructor(address owner_) Ownable(owner_) { }

function setCredContract(address credContract_) external onlyOwner {
    credContract = ICred(credContract_);
}
```

If `setCredContract()` is not called before other functions that use `credContract`, those functions will revert due to calling methods on a zero address. Furthermore, the ability to change the `credContract` address at any time can be exploited to manipulate the contract's behavior.

### Impact 
The contract could become unusable if `setCredContract()` is not called, and the owner could manipulate the contract's behavior by changing the `credContract` address.

### Recommendation

1. Initialize `credContract` in the constructor and ensure it is not a zero address:
```solidity
constructor(address owner_, address credContract_) Ownable(owner_) {
    require(credContract_ != address(0), "Invalid cred contract address");
    credContract = ICred(credContract_);
}
```

2. Prevent multiple calls to `setCredContract()` by checking if it has already been set:
```solidity
function setCredContract(address credContract_) external onlyOwner {
    require(address(credContract) == address(0), "Cred contract already set");
    require(credContract_ != address(0), "Invalid cred contract address");
    credContract = ICred(credContract_);
}
```

3. Add checks in functions that use `credContract` to ensure it has been set:
```solidity
function getPriceData(
    uint256 credId_,
    uint256 supply_,
    uint256 amount_,
    bool isSign_
)
    public
    view
    returns (uint256 price, uint256 protocolFee, uint256 creatorFee)
{
    require(address(credContract) != address(0), "Cred contract not set");
    // ... rest of the function
}
```




## [L-05] Potential Precision Loss in Reward Calculations

### Vulnerability Detail

The `handleRewardsAndGetValueSent()` function in the `PhiRewards` contract calculates rewards for various participants (artist, referral, verifier, and curator) based on a `quantity_` parameter. The calculations are performed using integer multiplication, which can lead to precision loss due to rounding errors when `quantity_` is not evenly divisible by the reward amounts.

The relevant code is as follows:

```solidity
function handleRewardsAndGetValueSent(
    uint256 artId_,
    uint256 credId_,
    uint256 quantity_,
    uint256 mintFee_,
    bytes calldata addressesData_,
    bool chainSync_
)
    external
    payable
{
    if (computeMintReward(quantity_, mintFee_) != msg.value) {
        revert InvalidDeposit();
    }

    depositRewards(
        artId_,
        credId_,
        addressesData_,
        quantity_ * (mintFee_ + artistReward),
        quantity_ * referralReward,
        quantity_ * verifierReward,
        quantity_ * curateReward,
        chainSync_
    );
}
```

The issue arises because the individual reward calculations (`quantity_ * reward`) are performed separately, which can lead to rounding errors. For example, if `quantity_` is chosen such that it causes rounding errors, the sum of individual rewards might not match the total `msg.value` sent, leading to discrepancies.

### Impact

Potential accumulation of undistributed funds in the contract due to rounding errors.

### Recommendation

Implement a more precise reward calculation system using fixed-point arithmetic to ensure precision in reward calculations. Additionally, consider implementing a dust collection mechanism to handle any remaining wei (dust) to ensure no funds are left unaccounted for.

```solidity
using FixedPoint for uint256;

uint256 constant PRECISION = 1e18;

function handleRewardsAndGetValueSent(
    uint256 artId_,
    uint256 credId_,
    uint256 quantity_,
    uint256 mintFee_,
    bytes calldata addressesData_,
    bool chainSync_
)
    external
    payable
{
    uint256 totalReward = computeMintReward(quantity_, mintFee_);
    require(totalReward == msg.value, "Invalid deposit amount");

    uint256 artistTotalReward = quantity_.mulFixed(artistReward.add(mintFee_), PRECISION);
    uint256 referralTotalReward = quantity_.mulFixed(referralReward, PRECISION);
    uint256 verifierTotalReward = quantity_.mulFixed(verifierReward, PRECISION);
    uint256 curateTotalReward = quantity_.mulFixed(curateReward, PRECISION);

    // Ensure no dust is left
    uint256 distributedReward = artistTotalReward.add(referralTotalReward).add(verifierTotalReward).add(curateTotalReward);
    require(distributedReward == totalReward, "Reward calculation mismatch");

    depositRewards(
        artId_,
        credId_,
        addressesData_,
        artistTotalReward,
        referralTotalReward,
        verifierTotalReward,
        curateTotalReward,
        chainSync_
    );
}

uint256 public dust;

function collectDust() external onlyOwner {
    uint256 amount = dust;
    dust = 0;
    payable(owner()).transfer(amount);
}
```