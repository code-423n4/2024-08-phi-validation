# Quality Assurance Report

The report consists of low-severity risk issues, governance risk issues arising from centralization vectors and a few non-critical issues in the end. The governance risks section describes scenarios in which the actors could misuse their powers.

 - [Low-severity issues](#low-severity-issues)
 - [Governance risks](#governance-risks)
   - [Roles/Actors in the system](#rolesactors-in-the-system)
   - [Powers of each role](#powers-of-each-role)
 - [Non-Critical issues](#non-critical-issues)

## Low-severity issues

## [L-01] Referrals can be exploited using alias addresses

**Summary:**
According to the [docs](https://docs.philand.xyz/explore-phi/phi-protocol/ecosystem/mint-referral#what-is-the-mint-referral-reward), the referral reward is intended to incentivize individuals who discover and promote a Cred NFT mint page. To earn the referral reward, someone must mint a Cred NFT they found through a link you have shared.

The issue in the codebase is that, even if you promote a cred NFT, anyone can use one of their addresses as the referral reward recipient. For example, bob owning address(A) promotes cred NFT to alice owning address(B). Alice though can use her alias address(C) as the recipient of the referral reward, thus earning it herself instead of bob who referred it to her.

This breaks the referral system since self referrals are still possible when rewards are deposited through the claimFromFactory() function of a PhiNFT1155.sol contract. To note, the referral address is validated through the signature claiming mechanism in the factory contract but is not validated through the merkle claiming mechanism. Due to this, self referrals would be possible when verification type is merkle.

Overall though, even the phiSignerAddress providing signatures from the frontend for signature claiming won't be able to protect against this since alice could simply connect her other address(C) to the site, take the referral link from it and supply it to her address(B) for referrals. 

**Proof of Concept**

1. Alice calls function [merkleClaim()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L352) in PhiFactory.sol. 
 - In `encodeData_` parameter, alice can encode the minter address as her address(B) and referral address as her address(C).
 - In the function, we can see that the referral address is not validated since it is dynamic, thus it would not make sense to include it in the merkle tree as well. This is contrary to the signature verification type where the referral address is validated with the signature. 
 - 
```solidity
    function merkleClaim(
        bytes32[] calldata proof_,
        bytes calldata encodeData_,
        MintArgs calldata mintArgs_,
        bytes32 leafPart_
    )
        external
        payable
        whenNotPaused
    {
        (address minter_, address ref_, uint256 artId_) = abi.decode(encodeData_, (address, address, uint256));
        PhiArt storage art = arts[artId_];

        ...

        if (
            !MerkleProofLib.verifyCalldata(
                proof_, credMerkleRootHash, keccak256(bytes.concat(keccak256(abi.encode(minter_, leafPart_))))
            )
        ) {
            revert InvalidMerkleProof();
        }
        
        ...

        _processClaim(
            artId_, minter_, ref_, art.credCreator, mintArgs_.quantity, leafPart_, mintArgs_.imageURI, msg.value
        );
    }
```

1. The call followingly routes as follows [_processClaim()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L723) => [claimFromFactory()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L164) (in PhiNFT1155 contract) => [handleRewardsAndGetValueSent()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L123) => [depositRewards()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L78)

2. In function depositRewards() below, we can observe the following:
 - Referral address is decoded i.e. address(C)
 - We do not enter the if block since in the first condition address(C) is not equal to address(B) and in the second condition address(C) is not address(0).
 - Due to this, the balance of address(C) is incremented with the referrer reward and alice successfully bypasses the self referral prevention mechanism that was set in the if condition i.e. `referral_ == minter_`

```solidity
    function depositRewards(
        uint256 artId_,
        uint256 credId_,
        bytes calldata addressesData_,
        uint256 artistTotalReward_,
        uint256 referralTotalReward_,
        uint256 verifierTotalReward_,
        uint256 curateTotalReward_,
        bool chainSync_
    )
        internal
    {
        (address minter_, address receiver_, address referral_, address verifier_) =
            abi.decode(addressesData_, (address, address, address, address));

        ...

        if (referral_ == minter_ || referral_ == address(0)) {
            artistTotalReward_ += referralTotalReward_;
            referralTotalReward_ = 0;
        } else if (referral_ != address(0)) {
            balanceOf[referral_] += referralTotalReward_;
        }
```

**Recommended Mitigation Steps**
A good way to mitigate this issue would be to check if the referral address supplied holds atleast 1 share of the cred NFT. This way we know that the referral address has bought a share of the cred NFT which is being promoted.

## [L-02] RewardsDeposit and Deposit events can be spam emitted with 0 value deposits through external function handleRewardsAndGetValueSent()

The handleRewardsAndGetValueSent() function is called from the PhiNFT1155.sol contracts deployed. This function currently is externally exposed which means events can be spam emitted by directly calling the function. These emission is possible by passing quantity as 0 and msg.value as 0. Performing this spam is easy on low fee chains that the protocol is currently planning to deploy on.

This is also an attack idea mentioned in the README - "Are there any potential issues with emitting events or executing functions when using Decent or relayer protocols?" and "Does the contract emit all necessary events correctly and include appropriate data?". Since we do not know how the events are going to be used by relayer protocols due to being out of scope, the issue is being submitted as QA. 
```solidity
    function depositRewards(
        uint256 artId_,
        uint256 credId_,
        bytes calldata addressesData_,
        uint256 artistTotalReward_,
        uint256 referralTotalReward_,
        uint256 verifierTotalReward_,
        uint256 curateTotalReward_,
        bool chainSync_
    )
        internal
    {
        ...

        bytes memory rewardsData;
        if (chainSync_ && address(curatorRewardsDistributor) != address(0)) {
            
            curatorRewardsDistributor.deposit{ value: curateTotalReward_ }(credId_, curateTotalReward_);
            ...
        } else {
            ...
        }

        ...

        emit RewardsDeposit(credData, minter_, receiver_, referral_, verifier_, rewardsData);
    }

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

The depositRewards() function from PhiRewards contract above calls the deposit() function in the CuratorRewardsDistributor.sol contract, which would also spam emit the Deposit events. 
```solidity
function deposit(uint256 credId, uint256 amount) external payable {
        if (!credContract.isExist(credId)) revert InvalidCredId();
        if (msg.value != amount) {
            revert InvalidValue(msg.value, amount);
        }
        balanceOf[credId] += amount;
        emit Deposit(credId, amount);
    }
```

**Solution:** Consider adding a check in the handleRewardsAndGetValueSent() and deposit() function, which disallows calls if quantity = 0 or amount = 0. 

## [L-03] Function distribute() in CuratorRewardsDistributor is prone to frontrunning

In the CuratorRewardsDistributor contract, any user can call the distribute() function through the frontend (as seen on the [demo site](https://base-sepolia.terminal.phiprotocol.xyz/cred/137)). The user would earn 1% (currently) of the total balance as the royalty reward.

The issue is that all distribute() function calls occurring from the frontend would be made by normal users, which a bot can frontrun by calling the distribute() function itself. This means normal users would never be able to receive royalty fees.

**Solution:** A good way to solve this is by implementing some minimum requirement. E.g.: Only users who have been holding shares of that credId for a certain duration would be able to call distribute(). 
```solidity
File: CuratorRewardsDistributor.sol
081:     function distribute(uint256 credId) external {
082:         if (!credContract.isExist(credId)) revert InvalidCredId();
083:         uint256 totalBalance = balanceOf[credId];
084:         
             ...

103:         uint256 royaltyfee = (totalBalance * withdrawRoyalty) / RATIO_BASE;
104:         
             ...
122: 
123:         balanceOf[credId] -= totalBalance;
124: 
125:         _msgSender().safeTransferETH(royaltyfee + distributeAmount - actualDistributeAmount);
126: 
127:         ...
131: 
132:         ...
135:     }
```

## [L-04] Malicious user can game curator rewards distribution model in certain conditions

**Context:**
The distribute() function in CuratorRewardsDistributor distributes curator rewards accumulated through different artId mints under a credId. The `balanceOf[credId]` is distributed to users who hold non-zero amount of shares of that credId.

**Issue:**
A malicious user could frontrun a distribute(credId) function call to buy a huge amount of shares of that credId. This would put the user into the curator addresses list that is eligible to receive the rewards.

If we check lines 114-115 in the snippet below, we can see that it uses the shares of a curator and divides it by the total number of shares to determine the % of distributeAmount to give to the curator. Since the malicious user frontran the distribute() call by buying a large amount of shares, the rewards would be diluted and distributed when the distribute() function call goes through. 

Through this, we can see that although the malicious user just bought the shares, the user is immediately eligibile for the curator rewards. Many malicious users could exploit this opportunity to earn a good chunk of distributeAmount.

To note, there is obviously the 10 minute share lock period that the malicious user will have to adhere to, during which some curators can sell to profit from the malicious user's buy that increased the price. Since there is a tradeoff introduced with the 10 minute delay, the net profit an attacker could make through this won't be much but the possibility still exists since previous curators may not sell.

**Solution:**
Introduce a minimum time period e.g. 10 minutes required before being eligible for curator reward distribution. 

```solidity
File: CuratorRewardsDistributor.sol
081:     function distribute(uint256 credId) external {
082:         if (!credContract.isExist(credId)) revert InvalidCredId();
083:         uint256 totalBalance = balanceOf[credId];
084:         
             ...

089:         address[] memory distributeAddresses = credContract.getCuratorAddresses(credId, 0, 0);
090:         uint256 totalNum;
091: 
092:         for (uint256 i = 0; i < distributeAddresses.length; i++) {
093:             totalNum += credContract.getShareNumber(credId, distributeAddresses[i]);
094:         }
095: 
096:         ...
099: 
100:         uint256[] memory amounts = new uint256[](distributeAddresses.length);
101:         
             ...
105: 
106:         // actualDistributeAmount is used to avoid rounding errors
107:         // amount[0] = 333 333 333 333 333 333
108:         // amount[1] = 333 333 333 333 333 333
109:         // amount[2] = 333 333 333 333 333 333
110:         uint256 actualDistributeAmount = 0;
111:         for (uint256 i = 0; i < distributeAddresses.length; i++) {
112:             address user = distributeAddresses[i];
113: 
114:             uint256 userAmounts = credContract.getShareNumber(credId, user);
115:             uint256 userRewards = (distributeAmount * userAmounts) / totalNum;
116: 
117:             if (userRewards > 0) {
118:                 amounts[i] = userRewards;
119:                 actualDistributeAmount += userRewards;
120:             }
121:         }
             ...
135:     }
```

## [L-05] Creator does not receive fees when supply = 0

Creator does not receive creator fees when supply = 0, which is for the initial case when a share is bought during cred creation. This is an issue since the msg.sender that receives the first share can be a party other than the creator supplied in the cred that is being created.

Overall, the value lost is negligible and poses a QA risk. 
```solidity
File: BondingCurve.sol
51:     function getPriceData(
52:         uint256 credId_,
53:         uint256 supply_,
54:         uint256 amount_,
55:         bool isSign_
56:     )
57:         public
58:         view
59:         returns (uint256 price, uint256 protocolFee, uint256 creatorFee)
60:     {
61:         (uint16 buyShareRoyalty, uint16 sellShareRoyalty) = credContract.getCreatorRoyalty(credId_);
62: 
63:         price = isSign_ ? getPrice(supply_, amount_) : getPrice(supply_ - amount_, amount_);
64: 
65:       
66:         protocolFee = _getProtocolFee(price);
67:         
68:         if (supply_ == 0) {
69:             creatorFee = 0;
70:             return (price, protocolFee, creatorFee);
71:         }
72:         uint16 royaltyRate = isSign_ ? buyShareRoyalty : sellShareRoyalty;
73:         creatorFee = (price * royaltyRate) / RATIO_BASE;
74:     }
```

## [L-06] Missing receiver address validation during artId creation in _initializePhiArt() could DOS future claiming

When an artId is being created, the _initializeArt() function is called as part of the process. In this function the receiver address is stored in the PhiArt struct. The issue is that this address is not validated to not be address(0).

```solidity
File: PhiFactory.sol
642:     function _initializePhiArt(PhiArt storage art, ERC1155Data memory createData_) private {
643:         (
644:             uint256 credId,
645:             address credCreator,
646:             string memory verificationType,
647:             uint256 credChainId,
648:             bytes32 merkleRootHash
649:         ) = abi.decode(createData_.credData, (uint256, address, string, uint256, bytes32));
650: 
651:         ...
658:         art.receiver = createData_.receiver;
659:         ...
```

Due to this, during merkle or signature claiming from factory, the call would revert. This is because the handleRewardsAndGetValueSent() function calls the depositRewards() function that expects the receiver address of the artist to be non-zero as seen [here](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L93).

```solidity
  function claimFromFactory(
        uint256 artId_,
        address minter_,
        address ref_,
        address verifier_,
        uint256 quantity_,
        bytes32 data_,
        string calldata imageURI_
    )
        external
        payable
        whenNotPaused
        onlyPhiFactory
    {
        ...

        IPhiRewards(payable(phiFactoryContract.phiRewardsAddress())).handleRewardsAndGetValueSent{ value: msg.value }(
            artId_, credId, quantity_, mintFee(tokenId_), addressesData_, credChainId == block.chainid
        );
    }
```

**Solution:**
Add the zero address check during artId creation i.e. in _initializePhiArt() function.

## [L-07] Missing whenNotPaused() modifier on updateRoyalties(), safeTransferFrom() and safeBatchTransferFrom() functions in PhiNFTERC1155.sol

When a PhiNFTERC1155.sol contract is paused by the owner i.e. the factory contract and thus the owner of the factory contract, it does not apply the pausedrestriction to the updateRoyalties() function and the two transfer functions. 

This occurs due to the missing whenNotPaused() modifier on them, which allows artist to update royalties even when contract is paused. Due to this, the NFT artist can still update royalties. This along with the fact that the NFT can be transferred using the two functions during the paused state can be a problem since NFT can continue to be traded in the external market and the artist can continue earning royalties through these trades

There does not seem to be a clear reason why a PhiNFT1155.sol contract would need to be paused other than to permanently halt creation and claiming of tokenIds. The team could maybe consider adding the whenNotPaused() modifier to the updateRoyalties() function and the transfer functions. 

Overall, it is highly unlikely a PhiNFT1155 contract would be paused and since the impact is not greater than medium, the issue is being marked as QA.
```solidity
File: PhiNFT1155.sol
201:     function updateRoyalties(
202:         uint256 tokenId_,
203:         RoyaltyConfiguration memory configuration
204:     )
205:         external
206:         onlyArtCreator(tokenId_)
207:     {
208:         _updateRoyalties(tokenId_, configuration);
209:     }
```

## [L-08] PhiNFT1155.sol contract will not be upgradeable contrary to documentation

According to the docs [here](https://github.com/code-423n4/2024-08-phi/blob/main/docs/overview.md#upgradeability) on upgradeability, the PhiNFT1155 contracts use the UUPS upgradeability pattern and can be upgraded by the owner.

Currently though, it is not possible to upgrade them since msg.sender has to be the owner, which would be the factory contract and the factory contract does not have a mechanism to upgrade contracts i.e. call the upgrade functionality in the UUPSUpgradeable contract inherited by PhiNFT1155.

It's not clear how the factory contract could have and support multiple implementation contracts of PhiNFT1155 (since many contracts can be created through function [createArt()](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L196) in the PhiFactory contract). If upgradeability needs to be supported, introduce a proxy for each PhiNFT1155 contract deployed and a mechanism upgrade it through the factory. If not, correct the documentation. 
```solidity
File: PhiNFT1155.sol
366:     function _authorizeUpgrade(address newImplementation) internal override onlyOwner {
367:         // This function is intentionally left empty to allow for upgrades
368:     }
```

## [L-09] Attacker can use function withdrawFor() to DOS users from withdrawing

Attacker can frontrun a user's withdraw() or withdrawWithSig() function full balance withdrawals with small withdrawFor() calls to grief users. For example, if user A tries to withdraw 100 tokens using function withdraw() or withdrawWithSig(), the attacker can frontrun and withdraw a 1 wei amount using function withdrawFor(). This would cause A's call to revert since now only 100 - 1 wei amount of tokens are withdrawable. 

Another issue that arises is that the withdrawFor() function defaults the destination to the "from" address. This is an issue since if the "from" address cannot deal with withdrawing msg.value (which is unlikely though) or if the user wanted to withdraw to another address, the tokens could be stuck in the "from" address contract or just simply sent to the "from" address EOA that was not supposed to receive the tokens. Due to this, an attacker is able to force a withdrawal to the "from" address. 

**Solution:**
Solution: Introduce an allowance system for function withdrawFor(). 
```solidity
File: RewardControl.sol
101:     function withdrawFor(address from, uint256 amount) external {
102:         _withdraw(from, from, amount);
103:     }
```

## [L-10] Royalties would round down to 0 when the unit of exchange has low decimals

Note: Low decimal tokens are in scope as per the [README](https://github.com/code-423n4/2024-08-phi#erc20-token-behaviors-in-scope).

In CreatorRoyaltiesControl.sol contract (which is inherited by PhiNFT1155.sol contract), rounding down of royalties is possible if salePrice is denominated in a token that has low decimals. 

For example, royaltybps = 10 (i.e. 0.1%) and salePrice = upto 999 (meaning upto 0.999 tokens for a token with two decimals). When we perform calculations, the royaltyAmount would be returned as 10 * 999 (max) / 10000 = 9990/10000 = 0. 

According to [EIP 2981](https://eips.ethereum.org/EIPS/eip-2981) - "If the royalty fee calculation results in a remainder, implementers MAY round up or round down to the nearest integer. For example, if the royalty fee is 10% and _salePrice is 999, the implementer can return either 99 or 100 for royaltyAmount, both are valid.". This is optional to the protocol as mentioned by the spec but since it could result in the royalties not being received by the artist, it is a valid issue to consider 

Note we can also see the IERC2981 inherited contract in CreatorRoyaltiesControl.sol which mentions - "Returns how much royalty is owed and to whom, based on a sale price that may be denominated in any unit of exchange. The royalty amount is denominated and should be paid in that same unit of exchange." This further proves that the contract is supposed to support and return royalties in any unit of exchange requested by an external marketplace, which isn't currently done. 
```solidity
File: CreatorRoyaltiesControl.sol
43:     function royaltyInfo(
44:         uint256 tokenId,
45:         uint256 salePrice
46:     )
47:         public
48:         view
49:         returns (address receiver, uint256 royaltyAmount)
50:     {
51:         RoyaltyConfiguration memory config = getRoyalties(tokenId);
52:         royaltyAmount = (config.royaltyBPS * salePrice) / ROYALTY_BPS_TO_PERCENT;
53:         receiver = config.royaltyRecipient;
54:     }
```

## [L-11] Consider using < instead of <= for expiry checks

Consider using < instead of <= on Line 625 below. From the frontend, we might just want the signature to be valid for the current block. So we just use block.timestamp but currently it requires expiresIn to be greater than block.timestamp.
```solidity
File: PhiFactory.sol
622:     function _validateArtCreationSignature(bytes calldata signedData_, bytes calldata signature_) private view {
623:         if (_recoverSigner(keccak256(signedData_), signature_) != phiSignerAddress) revert AddressNotSigned();
624:         (uint256 expiresIn_,,) = abi.decode(signedData_, (uint256, string, bytes));
625:         if (expiresIn_ <= block.timestamp) revert SignatureExpired();
626:         
627:     }
```

A similar instance also exists in the function signatureClaim() in the phiFactory.sol contract and createCred() function in the Cred.sol contract.

## [L-12] Old signatures supplied would become invalid and DOS art creation if phiSignerAddress is changed

During creation of art, we verify that the signedData i.e. credId and uri supplied is validated by the phiSignerAddress before providing the signature. This signature is then verified onchain as seen in the function below. The issue is that if the phiSignerAddress is updated through the setter, the latest signatures issues would cause a revert during art creation since Line 623 would evaluate to true.

Although this works as intended and users can simply take new signatures from the new phiSignerAddress, it is an edge case worth looking into to avoid unnecessary latest signature failures when a phiSignerAddress is changed. 

**Solution:** Before supplying signatures from the frontend, ensure the phiSignerAddress has not been updated onchain. If updated, supply signature through the frontend provided the new phiSignerAddress is configured. 
```solidity
File: PhiFactory.sol
622:     function _validateArtCreationSignature(bytes calldata signedData_, bytes calldata signature_) private view {
623:         if (_recoverSigner(keccak256(signedData_), signature_) != phiSignerAddress) revert AddressNotSigned();
624:         (uint256 expiresIn_,,) = abi.decode(signedData_, (uint256, string, bytes));
625:         if (expiresIn_ <= block.timestamp) revert SignatureExpired();
626:         
627:     }
```

## [L-13] Function signatureClaim() does not check if verification type is SIGNATURE

Unlike merkleClaim(), function signatureClaim() does not check if the verification type of the artId being minted is SIGNATURE or not. This is in case the function is directly called and not through the claim() function. 

Although I'm assuming the phiSignerAddress will be providing signatures for the `encodedData_` only for signature verification types, it would be good to add the check as done in merkleClaim(). 

```solidity
File: PhiFactory.sol
344:     function signatureClaim(
345:         bytes calldata signature_,
346:         bytes calldata encodeData_,
347:         MintArgs calldata mintArgs_
348:     )
349:         external
350:         payable
351:         whenNotPaused
352:     {
353:         (uint256 expiresIn_, address minter_, address ref_, address verifier_, uint256 artId_,, bytes32 data_) =
354:             abi.decode(encodeData_, (uint256, address, address, address, uint256, uint256, bytes32));
355: 
356:      
357:         if (expiresIn_ <= block.timestamp) revert SignatureExpired();
358:         if (_recoverSigner(keccak256(encodeData_), signature_) != phiSignerAddress) revert AddressNotSigned();
359:         
360:        
361:         _validateAndUpdateClaimState(artId_, minter_, mintArgs_.quantity);
362:         _processClaim(artId_, minter_, ref_, verifier_, mintArgs_.quantity, data_, mintArgs_.imageURI, msg.value);
363:       
365:         emit ArtClaimedData(artId_, "SIGNATURE", minter_, ref_, verifier_, arts[artId_].artAddress, mintArgs_.quantity);
366:     }
```

## [L-14] Missing whenNotPaused() modifier on function updateArtSettings()

Missing whenNotPaused() modifier on updateArtSettings() allows any artist to update their art settings when the contract is paused. Although there does not seem to be a risk currently, it should be disallowed if the factory itself is paused. 

```solidity
File: PhiFactory.sol
227:     function updateArtSettings(
228:         uint256 artId_,
229:         string memory url_,
230:         address receiver_,
231:         uint256 maxSupply_,
232:         uint256 mintFee_,
233:         uint256 startTime_,
234:         uint256 endTime_,
235:         bool soulBounded_,
236:         IPhiNFT1155Ownable.RoyaltyConfiguration memory configuration
237:     )
238:         external
239:         onlyArtCreator(artId_)
240:     {
```

## [L-15] A removed bonding curve would still be used by previously created creds

A bonding curve can be removed from the whitelist using the function removeFromWhitelist(). The issue is that previously created creds keep on using the old bonding curves that have been removed from the system.

It's not clear whether previous creds using the old bonding curves should continue to operate if a the curve is removed, due to which this is being submitted as a QA risk. 

```solidity
File: Cred.sol
164:     function addToWhitelist(address address_) external onlyOwner {
165:         curatePriceWhitelist[address_] = true;
166:         emit AddedToWhitelist(_msgSender(), address_);
167:     }
168: 
169:     /// @notice Removes an address from the whitelist.
170:     /// @param address_ The address to remove from the whitelist.
171:     function removeFromWhitelist(address address_) external onlyOwner {
172:         curatePriceWhitelist[address_] = false;
173:         emit RemovedFromWhitelist(_msgSender(), address_);
174:     }
```

## [L-16] Normal users are always prone to incurring a loss in Cred.sol due to MEV

Normal users are always prone to incurring a loss due to MEV (who the creators of a cred could themselves be or someone else). 

For example, user A places buy order, MEV bot frontruns buy order with its own buy order. Assuming user A is fine with price increase, the tx goes through. After 10 minutes of the share lock period, MEV bot sells shares with sell order. Unless victim user A is another MEV bot, normal users would always be incurring this loss since they would not be able to sell before the 10 minute period is over.

```solidity
File: Cred.sol
185:     function sellShareCred(uint256 credId_, uint256 amount_, uint256 minPrice_) public {
186:         _handleTrade(credId_, amount_, false, _msgSender(), minPrice_);
187:     }
```

## [L-17] Consider introducing a rate limit as to how often mintFee and soulBounded values can be changed.

Artist has the power to increase fees by frontrunning certain users. This could be used to favour some users over the others by charging them less and the others more.

Similarly, the soulBounded value can be changed by artist to favour certain users to transfer their NFTs in the same block. Following this, the artist can reset it back to true. 

Solution: Consider having some buffer period as to how often these values can be updated. 
```solidity
File: PhiFactory.sol
219:     function updateArtSettings(
220:         uint256 artId_,
221:         string memory url_,
222:         address receiver_,
223:         uint256 maxSupply_,
224:         uint256 mintFee_,
225:         uint256 startTime_,
226:         uint256 endTime_,
227:         bool soulBounded_,
228:         IPhiNFT1155Ownable.RoyaltyConfiguration memory configuration
229:     )
```

## Governance risks

### Roles/Actors in the system

1. PhiFactory
   - Contract owner
   - phiSignerAddress
   - protocolFeeDestination
2. PhiNFT1155
   - phiFactoryContract (Contract owner)
   - protocolFeeDestination
   - Artists
3. PhiRewards
   - Contract owner
4. CuratorRewardsDistributor
   - Contract owner
5. Cred
   - Contract owner
   - phiSignerAddress
   - protocolFeeDestination
6. BondingCurve
   - Contract owner

### Powers of each role

1. PhiFactory
   - Contract owner - Owner has access to pause() and unpause() functionality. A malicious scenario would be where the owner pauses the contract and renounces ownership. Similarly, the owner has the power to pause the PhiNFT1155 art contracts and perform the same behaviour. It also has the power to change the phiSignerAddress. If phiSignerAddress is changed maliciously, previous signatures would become invalid, which would temporarily halt any art creation and signature claiming until new signatures are provided. Some more core functions that the owner could misuse are setPhiRewardsAddress(), setErc1155ArtAddress(), setProtocolFeeDestination(), setProtocolFee() and setArtCreatFee().
   - phiSignerAddress - The phiSignerAddress provides signatures to create art for a credId and claiming for an artId. Importantly, it verifies the credData and claimData are correct. Specifically, for signature claiming it has the power to supply an incorrect verifier address that can receive the verifier rewards, effectively stealing from the actual verifier that was supposed to receive the fees. The same also applies for the referral rewards. Overall, it holds the power to the extent of potentially DOS claiming and art creation by not providing signatures for some users. 
   - protocolFeeDestination - This address, if a contract, can revert in its fallback function when it receives the fees through function _processClaim(). This could be done to favour certain users over others when claiming. 
2. PhiNFT1155
   - phiFactoryContract (Contract owner) - Ability to perform art creations, claims, pause and unpause the contract. Pausing and unpausing is indirectly influenced by the PhiFactory contract's owner. 
   - protocolFeeDestination - The protocolFeeDestination address receives the art create fees. It could revert (if a contract) in its fallback function when receiving the fees to deny some users from creating art for a credential i.e. credId.
   - Artists - Artists have the power to update royalties at any time. This is true even when the contract is paused. A malicious artist has the ability to set the royalty fee to a value greater than what is being read by external marketplaces. This could be achieved by frontrunning the external marketplaces calls to royaltyInfo(). Since there is no rate limit imposed on updating royalties, it allows an artist to misuse this functionality. 
3. PhiRewards
   - Contract owner - Owner has the ability to determine artist, referral, verifier and curator rewards. If curator reward is set to 0, there would be no tokens to distribute through function distribute() in CuratorRewardsDistributor.sol contract. The owner also has the ability to update CuratorRewardsDistributor.sol contract itself. This means that if the address of that contract is changed to a malicious contract address, the owner can slowly leak curator rewards without anyone noticing. 
4. CuratorRewardsDistributor
   - Contract owner - Owner has the ability to set the phiRewardsContract address and the withdrawRoyalty distributed to the msg.sender of the distribute() function as a royalty fee. Similar to the owner of PhiRewards contract, this owner can set the phiRewardsContract to a malicious contract address. When the depositBatch() function call goes through the distribute() function, it would send all the tokens to the malicious contract address. Secondly, the owner has the ability to set the withdrawRoyalty. This means it could update the withdrawRoyalty to a higher value than intended by the system and call the distribute() function for a credId in the same batch transaction. This could be used to always steal the royalty fees at a higher rate.
5. Cred
   - Contract owner - The owner has the ability to pause/unpause the contract at any time. It also has access to core functions setPhiSignerAddress(), setProtocolFeeDestination(), setProtocolFeePercent(), setPhiRewardsAddress(), addToWhitelist(), removeFromWhitelist() and finally to upgrade the contract itself. Most of these powers carry the risk as mentioned in the PhiFactory owner section above. But the most important point to note is the ability for protocol fee percent + creatorFee percent (i.e. buyShareRoyalty/sellShareRoyalty) to be greater than 10000 (in basis points). Due to this, we could underflow in function _handleTrade() [here](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L624) since the protocolFee and creatorFee summation would be greater than the price. This would prevent selling shares of all credIds and turn the contract into a honeypot. Consider ensuring in the setter function setProtocolFeePercent() that it is not greater than MAX_ROYALTY_RANGE of 5000, which is the max for setting creator fee buy share and sell share royalties when creating a cred. 
   - phiSignerAddress - Holds the power to allow/disallow cred creations and updates. It does not pose a significant risk in the updateCred() function but can when creating a cred with incorrect data [here](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L253). 
   - protocolFeeDestination - Can revert (if a contract) in its fallback function when fees are paid out to it through function _handleTrade().
6. BondingCurve
   - Contract owner - Owner has power to change cred contract variable to manipulate protocol fee percents and creator royalties (buyShareRoyalty/sellShareRoyalty) for creds that are currently being dynamically retrieved from Cred.sol. 

## Non-Critical issues

## [N-01] Remove redundant console2 foundry imports

The console2 imports seem to be have been used for testing but have not been removed. Consider removing it since they do not serve any use in production. 
```solidity
File: CuratorRewardsDistributor.sol
12: import { console2 } from "forge-std/console2.sol"; 

File: BondingCurve.sol
8: import { console2 } from "forge-std/console2.sol"; 
```

## [N-02] No function to update cred contract in CuratorRewardsDistributor.sol

In the CuratorRewardsDistributor.sol contract, there are only two functions which allow the owner to update the phi rewards contract and withdraw royalty. But there is no function to update the cred contract.

Although the cred implementation contract is upgradeable, if the proxy-implementation cred contracts itself need to be changed while keeping the distributor contracts the same, there won't be a way to update the cred contract address in CuratorRewardsDistributor.
```solidity
File: CuratorRewardsDistributor.sol
51:     function updatePhiRewardsContract(address phiRewardsContract_) external onlyOwner {
52:         if (phiRewardsContract_ == address(0)) {
53:             revert InvalidAddressZero();
54:         }
55:         phiRewardsContract = IPhiRewards(phiRewardsContract_);
56:         emit PhiRewardsContractUpdated(phiRewardsContract_);
57:     }
58: 
59:     function updateRoyalty(uint256 newRoyalty_) external onlyOwner {
60:         if (newRoyalty_ > MAX_ROYALTY_RANGE) {
61:             revert InvalidRoyalty(newRoyalty_);
62:         }
63:         withdrawRoyalty = newRoyalty_;
64:         emit RoyaltyUpdated(newRoyalty_);
65:     }
```

## [N-03] Remove redundant ReentrancyGuardUpgradeable inheritance 

The ReentrancyGuardUpgradeable nonReentrant is not used in PhiNFT1155.sol contract. Consider removing the inheritance. 
```solidity
File: PhiNFT1155.sol
21: contract PhiNFT1155 is
22:     Initializable,
23:     UUPSUpgradeable,
24:     ERC1155SupplyUpgradeable,
25:     ReentrancyGuardUpgradeable, 
```

## [N-04] Remove redundant protocolFeeDestination() call

The below code is present in the initialize() function in the PhiNFT1155.sol contract. This function call to retrieve the protocolFeeDestination is not necessary since it is already supplied as a parameter to the initialize() function.

Since initializations only occur from the phiFactoryContract, the parameter would represent the latest protocolFeeDestination. 
```solidity
File: PhiNFT1155.sol
122:         protocolFeeDestination = phiFactoryContract.protocolFeeDestination();
```

## [N-05] Consider reverting with a reason instead of running into underflow error

The function createArtFromFactory() is called when an artId is being created through the phiFactory contract. The issue is that if enough msg.value is not supplied to cover artFee, we revert with an underflow error. 

It would be better to revert with a reason e.g. error NotEnoughFees(). 
```solidity
File: PhiNFT1155.sol
139:     function createArtFromFactory(uint256 artId_) external payable onlyPhiFactory whenNotPaused returns (uint256) {
140:         
154:         ...

155:         if ((msg.value - artFee) > 0) {
156:            
157:          
158:             _msgSender().safeTransferETH(msg.value - artFee);
159:         }
160: 
161:         return createdTokenId;
162:     }
```

## [N-06] Consider removing redundant conditions from functions safeTransferFrom() and safeBatchTransferFrom() in PhiNFT1155.sol contract

Remove first condition `from_ != address(0)` from checks below since it is already checked in the internal function calls. 
```solidity
File: PhiNFT1155.sol
318:     function safeTransferFrom(
319:         address from_,
320:         address to_,
321:         uint256 id_,
322:         uint256 value_,
323:         bytes memory data_
324:     )
325:         public
326:         override
327:     {   
328:         
329:         if (from_ != address(0) && soulBounded(id_)) revert TokenNotTransferable();
330:         address sender = _msgSender();
331:         ...
336:     }
337: 
338:  
344:     function safeBatchTransferFrom(
345:         address from_,
346:         address to_,
347:         uint256[] memory ids_,
348:         uint256[] memory values_,
349:         bytes memory data_
350:     )
351:         public
352:         override
353:     {
354:        
355:         for (uint256 i; i < ids_.length; i++) {
356:             if (from_ != address(0) && soulBounded(ids_[i])) revert TokenNotTransferable();
357:         }
358:         ...
363:     }
```

## [N-07] Consider using an array of comments for function depositBatch()

The function depositBatch() does not allow each deposit to have a separate comment. Consider passing a comments array as well. There does not seem to be a problem with the current contracts in-scope but support can be extended if the function is used in future contract integrations. 

```solidity
File: RewardControl.sol
56:     function depositBatch(
57:         address[] calldata recipients,
58:         uint256[] calldata amounts,
59:         bytes4[] calldata reasons,
60:         string calldata comment
61:     )
62:         external
63:         payable
64:     {
```

## [N-08] Consider removing redundant function withdraw() from PhiFactory.sol

The function withdraw() is not required since the protocol fees are currently sent to the protocolFeeDestination correctly. 
```solidity
File: PhiFactory.sol
830:     function withdraw() external onlyOwner {
831:         protocolFeeDestination.safeTransferETH(address(this).balance);
832:     }
```

## [N-09] Remove TODOs from the code

The artId check mentioned by the TODO comment on Line 583 has already been implemented, consider removing it.
```solidity
File: PhiFactory.sol
581:     function createERC1155Internal(uint256 newArtId, ERC1155Data memory createData_) internal returns (address) {
582:        
583:         // Todo check if artid exists, if artid exists, reverts
```

## [N-10] Price limits array should be checked for equal length

In the _validateAndCalculateBatch() function, we check the `credIds_` and `amounts_` arrays to be of equal length but this is not done for `priceLimits_` as well.
```solidity
File: Cred.sol
832:     function _validateAndCalculateBatch(
833:         uint256[] calldata credIds_,
834:         uint256[] calldata amounts_,
835:         uint256[] calldata priceLimits_,
836:         bool isBuy
837:     )
838:         internal
839:         view
840:         returns (
841:             uint256 totalAmount,
842:             uint256[] memory prices,
843:             uint256[] memory protocolFees,
844:             uint256[] memory creatorFees
845:         )
846:     {
847:        
848:         uint256 length = credIds_.length;
849:         if (length != amounts_.length) {
850:             revert InvalidArrayLength();
851:         }
```