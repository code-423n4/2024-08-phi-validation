| Index | Title                                                                                                           |
| ----- | --------------------------------------------------------------------------------------------------------------- |
| L-01  | Withdrawals would fail for some users                                           |
| L-02  | Signatures should not be usable by the deadlines                                                       |
| L-03  | withdrawFor can be used to lock some users asset |
| L-04  | Remove redundant check when withdrawing                                                                                      |
| L-05  | Distributions can be stolen from honest users                                   |
| L-06  | Attackers will steal tokens via frequent distributions                                                                                 |
| L-07  | Setters don't have equality checkers
| L-08  | Fix inconsistency between interface and contract
| L-09  | Public Internal Functions Can Be Called Externally
| L-10  | Lack of Input Validation for Empty Arrays in `batchClaim` Function
| L-11  | Typos/Misspellings

## L-1 Withdrawals would fail for some users
### Impact

This vulnerability easily impacts Phi's ability to validate signatures for counterfactual ERC-4337 accounts, limiting the usability for users of certain wallets that rely on AA, leading to the limitation of functionalities in the protocol, since all operations that need the signatures attached to the typehashes wouldn't work for some set of users, i.e the availability of these functions is impacted.


This then means that some of the protocol would be unable to validate real signatures even from users who are expected to integrate with the protocol since the revert from `PermitLib#requireSignature()` would bubble back up to all instances where it's been used across protocol, for e.g in [RewardControl#withdrawWithSig](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L102).
### Proof of Concept

[EIP 1271](https://eips.ethereum.org/EIPS/eip-1271) is being implemented in protocol, albeit via the `SignatureCheckerUpgradeable` inherited from openzeppelin which allows contracts to sign messages and works great in tandem with EIP 4337 (account abstraction).

For ERC-4337, the account is not deployed until the first UserOp is mined, previous to this, the account exists "counterfactually" — it has an address even though it's not really deployed. Usually, this is great since we can use the counterfactual address to receive assets without deploying the account first. Now, not being able to sign messages from counterfactual contracts/accounts has always been a limitation of [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) since one can't call the `_isValidSignature()` function on them.

Now in Phi there is an option to withdraw vua signature: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L100-L110

```solidity
    function withdrawWithSig(address from, address to, uint256 amount, uint256 deadline, bytes calldata sig) external {
        if (block.timestamp > deadline) revert DeadlineExpired();
        if (!_verifySignature(from, to, amount, nonces[from], deadline, sig)) revert InvalidSignature();

        unchecked {
            ++nonces[from];
        }

        _withdraw(from, to, amount);
    }

```


https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L146-L161

```solidity
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

Which calls the below via `SignatureCheckerLib`

```solidity
    function isValidSignatureNowCalldata(address signer, bytes32 hash, bytes calldata signature)
        internal
        view
        returns (bool isValid)
    {
        /// @solidity memory-safe-assembly
        assembly {
            // Clean the upper 96 bits of `signer` in case they are dirty.
            for { signer := shr(96, shl(96, signer)) } signer {} {
                let m := mload(0x40)
                mstore(0x00, hash)
                if eq(signature.length, 64) {
                    let vs := calldataload(add(signature.offset, 0x20))
                    mstore(0x20, add(shr(255, vs), 27)) // `v`.
                    mstore(0x40, calldataload(signature.offset)) // `r`.
                    mstore(0x60, shr(1, shl(1, vs))) // `s`.
                    let t :=
                        staticcall(
                            gas(), // Amount of gas left for the transaction.
                            1, // Address of `ecrecover`.
                            0x00, // Start of input.
                            0x80, // Size of input.
                            0x01, // Start of output.
                            0x20 // Size of output.
                        )
                    // `returndatasize()` will be `0x20` upon success, and `0x00` otherwise.
                    if iszero(or(iszero(returndatasize()), xor(signer, mload(t)))) {
                        isValid := 1
                        mstore(0x60, 0) // Restore the zero slot.
                        mstore(0x40, m) // Restore the free memory pointer.
                        break
                    }
                }
                if eq(signature.length, 65) {
                    mstore(0x20, byte(0, calldataload(add(signature.offset, 0x40)))) // `v`.
                    calldatacopy(0x40, signature.offset, 0x40) // `r`, `s`.
                    let t :=
                        staticcall(
                            gas(), // Amount of gas left for the transaction.
                            1, // Address of `ecrecover`.
                            0x00, // Start of input.
                            0x80, // Size of input.
                            0x01, // Start of output.
                            0x20 // Size of output.
                        )
                    // `returndatasize()` will be `0x20` upon success, and `0x00` otherwise.
                    if iszero(or(iszero(returndatasize()), xor(signer, mload(t)))) {
                        isValid := 1
                        mstore(0x60, 0) // Restore the zero slot.
                        mstore(0x40, m) // Restore the free memory pointer.
                        break
                    }
                }
                mstore(0x60, 0) // Restore the zero slot.
                mstore(0x40, m) // Restore the free memory pointer.

                let f := shl(224, 0x1626ba7e)
                mstore(m, f) // `bytes4(keccak256("isValidSignature(bytes32,bytes)"))`.
                mstore(add(m, 0x04), hash)
                let d := add(m, 0x24)
                mstore(d, 0x40) // The offset of the `signature` in the calldata.
                mstore(add(m, 0x44), signature.length)
                // Copy the `signature` over.
                calldatacopy(add(m, 0x64), signature.offset, signature.length)
                // forgefmt: disable-next-item
                isValid := and(
                    // Whether the returndata is the magic value `0x1626ba7e` (left-aligned).
                    eq(mload(d), f),
                    // Whether the staticcall does not revert.
                    // This must be placed at the end of the `and` clause,
                    // as the arguments are evaluated from right to left.
                    staticcall(
                        gas(), // Remaining gas.
                        signer, // The `signer` address.
                        m, // Offset of calldata in memory.
                        add(signature.length, 0x64), // Length of calldata in memory.
                        d, // Offset of returndata.
                        0x20 // Length of returndata to write.
                    )
                )
                break
            }
        }
    }

```

Which practically means that Phi will fail to validate signatures for users of notorious wallets/projects, including Safe, Sequence, and Argent supporting ERC1271, but also ERC4337 wallets, even though a clear intention has been made to support signatures by EIP1271 compliant wallets, as confirmed by using the Eip-1271 method of validating signatures.

Crux of the issue is the fact that protocol is taking responsibility to check the validity of signatures, but partially failing to trigger signature validation signatures for a group of wallets from a provider since _(the validation will succeed if the ERC4337 wallet is deployed)_ and given that the protocol decided to support contract-based wallets (that support ERC4337) and implement ERC1271, one could assume that they "inherit" the possibility from the wallet providers to support ERC4337 too.


### Recommended Mitigation Steps

[EIP-6492](https://eips.ethereum.org/EIPS/eip-6492) solves this issue. The author of the EIP already implemented ERC-6492 in a universal library which verifies 6492, 712, and 1271 sigs with this pull request.

ERC6492 assumes that the signing contract will normally be a contract wallet, but it could be any contract that implements [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) and is deployed counterfactually.

- If the contract is deployed, produce a normal [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) signature
- If the contract is not deployed yet, wrap the signature as follows: `concat(abi.encode((create2Factory, factoryCalldata, originalERC1271Signature), (address, bytes, bytes)), magicBytes)`

Phi could adopt the [reference-implementation](https://eips.ethereum.org/EIPS/eip-6492#reference-implementation) as stated in the EIP and delegate the responsibility of supporting counterfactual signatures to the wallets themselves, and this works because, as defined in the EIP, the wallet should return the magic value in both cases.
## L-2 Signatures should ot be usable by the deadlines

### Proof of Concept

Link to code snippet: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L100-L110

```solidity
    function withdrawWithSig(address from, address to, uint256 amount, uint256 deadline, bytes calldata sig) external {
        if (block.timestamp > deadline) revert DeadlineExpired();
        if (!_verifySignature(from, to, amount, nonces[from], deadline, sig)) revert InvalidSignature();

        unchecked {
            ++nonces[from];
        }

        _withdraw(from, to, amount);
    }

```

Issue here is the fact that at the deadlines the signatures still pass since the checks are not inclusive.

### Impact
QA
### Recommended Mitigation Steps
Make the `> deadline` check inclusive.
## L-3 `withdrawFor` can be used to lock some users asset
### Impact

Rewards could be lost/stuck for some users.

### Proof of Concept

Link to code snippet: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L96-L98

```solidity
    function withdrawFor(address from, uint256 amount) external {
        _withdraw(from, from, amount);
    }
```

This function is used to withdraw rewards on behalf of an address, issue here however is the fact that the function is publicly avaialby to anyone, now note that [from here](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L146-L161), i.e the signature verification logic we can see that Phi expects smart contracts/wallets to integrate with it's protocol which is why `SignatureCheckerLib.isValidSignatureNowCalldata()` is used to verify the signatures, which  also the `ERC1271` to check for contract, see the implementation here:


```solidity
    function isValidSignatureNowCalldata(address signer, bytes32 hash, bytes calldata signature)
        internal
        view
        returns (bool isValid)
    {
        /// @solidity memory-safe-assembly
        assembly {
            // Clean the upper 96 bits of `signer` in case they are dirty.
            for { signer := shr(96, shl(96, signer)) } signer {} {
                let m := mload(0x40)
                mstore(0x00, hash)
                if eq(signature.length, 64) {
                    let vs := calldataload(add(signature.offset, 0x20))
                    mstore(0x20, add(shr(255, vs), 27)) // `v`.
                    mstore(0x40, calldataload(signature.offset)) // `r`.
                    mstore(0x60, shr(1, shl(1, vs))) // `s`.
                    let t :=
                        staticcall(
                            gas(), // Amount of gas left for the transaction.
                            1, // Address of `ecrecover`.
                            0x00, // Start of input.
                            0x80, // Size of input.
                            0x01, // Start of output.
                            0x20 // Size of output.
                        )
                    // `returndatasize()` will be `0x20` upon success, and `0x00` otherwise.
                    if iszero(or(iszero(returndatasize()), xor(signer, mload(t)))) {
                        isValid := 1
                        mstore(0x60, 0) // Restore the zero slot.
                        mstore(0x40, m) // Restore the free memory pointer.
                        break
                    }
                }
                if eq(signature.length, 65) {
                    mstore(0x20, byte(0, calldataload(add(signature.offset, 0x40)))) // `v`.
                    calldatacopy(0x40, signature.offset, 0x40) // `r`, `s`.
                    let t :=
                        staticcall(
                            gas(), // Amount of gas left for the transaction.
                            1, // Address of `ecrecover`.
                            0x00, // Start of input.
                            0x80, // Size of input.
                            0x01, // Start of output.
                            0x20 // Size of output.
                        )
                    // `returndatasize()` will be `0x20` upon success, and `0x00` otherwise.
                    if iszero(or(iszero(returndatasize()), xor(signer, mload(t)))) {
                        isValid := 1
                        mstore(0x60, 0) // Restore the zero slot.
                        mstore(0x40, m) // Restore the free memory pointer.
                        break
                    }
                }
                mstore(0x60, 0) // Restore the zero slot.
                mstore(0x40, m) // Restore the free memory pointer.

                let f := shl(224, 0x1626ba7e)
                mstore(m, f) // `bytes4(keccak256("isValidSignature(bytes32,bytes)"))`.
                mstore(add(m, 0x04), hash)
                let d := add(m, 0x24)
                mstore(d, 0x40) // The offset of the `signature` in the calldata.
                mstore(add(m, 0x44), signature.length)
                // Copy the `signature` over.
                calldatacopy(add(m, 0x64), signature.offset, signature.length)
                // forgefmt: disable-next-item
                isValid := and(
                    // Whether the returndata is the magic value `0x1626ba7e` (left-aligned).
                    eq(mload(d), f),
                    // Whether the staticcall does not revert.
                    // This must be placed at the end of the `and` clause,
                    // as the arguments are evaluated from right to left.
                    staticcall(
                        gas(), // Remaining gas.
                        signer, // The `signer` address.
                        m, // Offset of calldata in memory.
                        add(signature.length, 0x64), // Length of calldata in memory.
                        d, // Offset of returndata.
                        0x20 // Length of returndata to write.
                    )
                )
                break
            }
        }
    }

```

Coupling these windows however, this then allows for a simple griefing window that locks a user's asset.

- Consider a smart contract integrates with Phi freqquently.
- Due to the native logic (code), it can't handle the rewards primarily and normally have to query the normal `withdraw(address to, uint256 amount)` in order to [send the rewards to the destined `to`](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L93).
>NB: The above can be due to any reason, be it from as simple as this contract just lacks this functionality since they can just query the normal `withdraw()`, to even more complex cases where we have the contract receiving the rewards to have some complex logic on deciding who among its users would be the recipient of the reward for that duration, etc. Also we'd assume this is a viable scenario since no restriction was place on who/what type of contract can integrate.
- A griefer can now at no cost whatsoever, just either frontrun the contract's attempt at withdrawing rewards to a different `to` address by calling `withdrawFrom()` which then locks the funds in the contract and this also doesn't neceassirly need to be a frontrun since the griefer can just query the function whenever some rewardds are accrued.


### Recommended Mitigation Steps
Consider not allowing a public withdrawal logic availabkle to  everyone, or make every integrator provide their prefferred primary reward address and then during withdrawals if `withdrawFor()` is used get the primary reward address from the mapping and send the reward to it. Or do not allow contracts/wallets to integrate (unadvisable).
## L-4 Remove redundant check when withdrawing
### Impact
QA, ineffective code.

### Proof of Concept

Link to code snippet: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L122-L139

```solidity
    function _withdraw(address from, address to, uint256 amount) internal {
        if (to == address(0)) revert InvalidAddressZero();

        uint256 balance = balanceOf[from];
        if (amount == FULL_BALANCE) {
            amount = balance;
        }

        if (amount > balance) revert InvalidAmount();

        unchecked {
            balanceOf[from] = balance - amount;
        }

        emit Withdraw(from, to, amount);

        to.safeTransferETH(amount);
    }
```

This is the internal function that helps with withdrawals, now there is an implementation that allows for the direct withdrawal the full balance, i.e by passing `0`, see https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L20

```solidity
    uint256 private constant FULL_BALANCE = 0;

```
Issue however is the fact that after [this block](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L126-L129) there is still a check if `(amount > balance)` this would be redundant though in the case where `FULL_BALANCE` is passed.
### Recommended Mitigation Steps
Break the code blocks into two, in the case `FULL_BALANCE` is passed do not check and in the case `FULL_BALANCE` is not passed then do otherwise.
## L-5 Distributions can be stolen from honest users
### Impact
Honest users would have their rewards stolen from them, since the attacker just front runs and receive the difference here `(royaltyfee + distributeAmount - actualDistributeAmount)`

### Proof of Concept

Link to code snippet: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L77-L130

```solidity
    function distribute(uint256 credId) external {
        if (!credContract.isExist(credId)) revert InvalidCredId();
        uint256 totalBalance = balanceOf[credId];
        if (totalBalance == 0) {
            revert NoBalanceToDistribute();
        }

        address[] memory distributeAddresses = credContract.getCuratorAddresses(credId, 0, 0);
        uint256 totalNum;

        for (uint256 i = 0; i < distributeAddresses.length; i++) {
            totalNum += credContract.getShareNumber(credId, distributeAddresses[i]);
        }

        if (totalNum == 0) {
            revert NoSharesToDistribute();
        }

        uint256[] memory amounts = new uint256[](distributeAddresses.length);
        bytes4[] memory reasons = new bytes4[](distributeAddresses.length);

        uint256 royaltyfee = (totalBalance * withdrawRoyalty) / RATIO_BASE;
        uint256 distributeAmount = totalBalance - royaltyfee;

        // actualDistributeAmount is used to avoid rounding errors
        // amount[0] = 333 333 333 333 333 333
        // amount[1] = 333 333 333 333 333 333
        // amount[2] = 333 333 333 333 333 333
        uint256 actualDistributeAmount = 0;
        for (uint256 i = 0; i < distributeAddresses.length; i++) {
            address user = distributeAddresses[i];

            uint256 userAmounts = credContract.getShareNumber(credId, user);
            uint256 userRewards = (distributeAmount * userAmounts) / totalNum;

            if (userRewards > 0) {
                amounts[i] = userRewards;
                actualDistributeAmount += userRewards;
            }
        }

        balanceOf[credId] -= totalBalance;

        _msgSender().safeTransferETH(royaltyfee + distributeAmount - actualDistributeAmount);

        //slither-disable-next-line arbitrary-send-eth
        phiRewardsContract.depositBatch{ value: actualDistributeAmount }(
            distributeAddresses, amounts, reasons, "deposit from curator rewards distributor"
        );

        emit RewardsDistributed(
            credId, _msgSender(), royaltyfee + distributeAmount - actualDistributeAmount, distributeAmount, totalBalance
        );
    }
```

This function is used to distribute the credit balance, in the case when `(totalBalance != 0)`, now the logic also includes sending back the caller the difference that occurs due to integer division, i.e https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L119-L121

```solidity

        _msgSender().safeTransferETH(royaltyfee + distributeAmount - actualDistributeAmount);

```

This however then opens an MEV bug window, where malicious tech savvy users would then easily front run honest user who are attempting to distribute and are actively tracking the balance to see when it's liquidatable.

### Recommended Mitigation Steps
Consider using a commmit-reveal logic for this
## L-6 Attackers will steal tokens via frequent distributions
### Impact
Unfair distribution for intended participants if the attacker backruns most of the minute deposits

### Proof of Concept

Link to code snippet: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L77-L130

```solidity
    function distribute(uint256 credId) external {
        if (!credContract.isExist(credId)) revert InvalidCredId();
        uint256 totalBalance = balanceOf[credId];
        if (totalBalance == 0) {
            revert NoBalanceToDistribute();
        }

        address[] memory distributeAddresses = credContract.getCuratorAddresses(credId, 0, 0);
        uint256 totalNum;

        for (uint256 i = 0; i < distributeAddresses.length; i++) {
            totalNum += credContract.getShareNumber(credId, distributeAddresses[i]);
        }

        if (totalNum == 0) {
            revert NoSharesToDistribute();
        }

        uint256[] memory amounts = new uint256[](distributeAddresses.length);
        bytes4[] memory reasons = new bytes4[](distributeAddresses.length);

        uint256 royaltyfee = (totalBalance * withdrawRoyalty) / RATIO_BASE;
        uint256 distributeAmount = totalBalance - royaltyfee;

        // actualDistributeAmount is used to avoid rounding errors
        // amount[0] = 333 333 333 333 333 333
        // amount[1] = 333 333 333 333 333 333
        // amount[2] = 333 333 333 333 333 333
        uint256 actualDistributeAmount = 0;
        for (uint256 i = 0; i < distributeAddresses.length; i++) {
            address user = distributeAddresses[i];

            uint256 userAmounts = credContract.getShareNumber(credId, user);
            uint256 userRewards = (distributeAmount * userAmounts) / totalNum;

            if (userRewards > 0) {
                amounts[i] = userRewards;
                actualDistributeAmount += userRewards;
            }
        }

        balanceOf[credId] -= totalBalance;

        _msgSender().safeTransferETH(royaltyfee + distributeAmount - actualDistributeAmount);

        //slither-disable-next-line arbitrary-send-eth
        phiRewardsContract.depositBatch{ value: actualDistributeAmount }(
            distributeAddresses, amounts, reasons, "deposit from curator rewards distributor"
        );

        emit RewardsDistributed(
            credId, _msgSender(), royaltyfee + distributeAmount - actualDistributeAmount, distributeAmount, totalBalance
        );
    }
```

This function is used to distribute the credit balance in the case when `(totalBalance != 0)`, now the logic also includes sending back the caller of the function  the difference that occurs due to integer division, i.e https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L119-L121

```solidity

        _msgSender().safeTransferETH(royaltyfee + distributeAmount - actualDistributeAmount);

```

Issue however is that there is no minimum balance requirement which allows this [check](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L80-L82) to be sidestepped no matter how small the accumulated balance is.

Now since [deposits](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L68) are going to be quite minute and accumulate over time, this then allows an attacker to always backrun  blocks where `(totalBalance != 0)`, i.e after [deposits from the rewarder contract](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L110) and in the same case where the round down that occurs [here](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L98-L119) would be massive for the intended distributors,  `~0` if deposited amount is quite small and then have the rewards sent to them via [CuratorRewardsDistributor.sol#L120](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/CuratorRewardsDistributor.sol#L120-L121).

### Recommended Mitigation Steps
Consider having a minimum balance being met before accepting distributions, i.e change this check to:

```diff
-        if (totalBalance == 0) {
+        if (totalBalance < minBalanceBeforeDistribute) {
            revert NoBalanceToDistribute();
        }
```
## L-7 Setters don't have equality checkers
### Impact

QA

### Proof of Concept

Link to code snippet: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/reward/PhiRewards.sol#L34-L64

```solidity
    /*//////////////////////////////////////////////////////////////
                            SETTER FUNCTIONS
    //////////////////////////////////////////////////////////////*/
    /// @notice Update artist reward amount
    /// @param newArtistReward_ New artist reward amount
    function updateArtistReward(uint256 newArtistReward_) external onlyOwner {
        artistReward = newArtistReward_;
        emit ArtistRewardUpdated(newArtistReward_);
    }

    /// @notice Update referral reward amount
    /// @param newReferralReward_ New referral reward amount
    function updateReferralReward(uint256 newReferralReward_) external onlyOwner {
        referralReward = newReferralReward_;
        emit ReferralRewardUpdated(newReferralReward_);
    }

    /// @notice Update verify reward amount
    /// @param newVerifyReward_ New verify reward amount
    function updateVerifierReward(uint256 newVerifyReward_) external onlyOwner {
        verifierReward = newVerifyReward_;
        emit VerifierRewardUpdated(newVerifyReward_);
    }

    /// @notice Update curate reward amount
    /// @param newCurateReward_ New curate reward amount
    function updateCurateReward(uint256 newCurateReward_) external onlyOwner {
        curateReward = newCurateReward_;
        emit CurateRewardUpdated(newCurateReward_);
    }

```



All these functions are used as setters, however there are no checks that `the current value being set != the previously stored value`.


### Recommended Mitigation Steps

Consider applying equality checkers.


## L-8 Fix inconsistency between interface and contract
### Impact
Confusing code.

### Proof of Concept

Link to code snippet: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/RewardControl.sol#L96-L98

```solidity
    function withdrawFor(address from, uint256 amount) external {
        _withdraw(from, from, amount);
    }
```

In the interface it's defined as: https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/interfaces/IRewards.sol#L80-L84

```solidity
    /// @notice Withdraws rewards on behalf of an address
    /// @param to Address to withdraw for
    /// @param amount Amount to withdraw (0 for total balance)
    function withdrawFor(address to, uint256 amount) external;

```

Evidetly, there is a mismatch between the parameters in the interface and the contract

### Recommended Mitigation Steps
Apply these changes:
```diff
-    function withdrawFor(address from, uint256 amount) external {
+    function withdrawFor(address to, uint256 amount) external {
-        _withdraw(from, from, amount);
+        _withdraw(to, to, amount);
    }
```

#
## L-09 Public Internal Functions Can Be Called Externally
### Impact
The `_addCredIdPerAddress` and `_removeCredIdPerAddress` functions are marked as `public`, allowing them to be called externally. These functions are intended to be internal helper functions, as indicated by their naming convention (prefixed with an underscore). Allowing external access to these functions can lead to unintended manipulation of the contract's internal state, potentially causing unexpected behavior or security vulnerabilities.
- State Manipulation: External actors could add or remove `credId` entries for any address, bypassing the intended logic and checks that should govern these operations.
### Proof of concept
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L685

```solidity
function _addCredIdPerAddress(uint256 credId_, address sender_) public {
```
- An external user calls `_addCredIdPerAddress` with arbitrary `credId` and `sender_` values.
- The function adds the `credId` to the `_credIdsPerAddress` mapping for the specified `sender_`, even if the `sender_` is not supposed to have that `credId`
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L695

```solidity
 function _removeCredIdPerAddress(uint256 credId_, address sender_) public {
```
- Similarly, an external user could call`_removeCredIdPerAddress` to remove a `credId` from an address, potentially disrupting the contract's logic.
### Recommendation
- Change the visibility of the `_addCredIdPerAddress` and `_removeCredIdPerAddress` functions from `public` to `internal`. This will restrict access to these functions, ensuring they can only be called from within the contract or its derived contracts, as intended.
```
function _addCredIdPerAddress(uint256 credId_, address sender_) internal {
    // Function implementation...
}

function _removeCredIdPerAddress(uint256 credId_, address sender_) internal {
    // Function implementation...
}
```
## L-10 Lack of Input Validation for Empty Arrays in `batchClaim` Function
### Impact
The `batchClaim` function does not validate whether the input arrays `encodeDatas_` and `ethValue_` are empty. This oversight allows transactions with empty arrays to be processed, potentially leading to:
- Gas waste on transactions that perform no actual claims.
- Potential misuse in complex scenarios where other parts of the system assume a successful batchClaim always results in claims being processed.
### Proof of concept
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L308-L321
```
function batchClaim(bytes[] calldata encodeDatas_, uint256[] calldata ethValue_) external payable {
    if (encodeDatas_.length != ethValue_.length) revert ArrayLengthMismatch();

    // calc claim fee
    uint256 totalEthValue;
    for (uint256 i = 0; i < encodeDatas_.length; i++) {
        totalEthValue = totalEthValue + ethValue_[i];
    }
    if (msg.value != totalEthValue) revert InvalidEthValue();

    for (uint256 i = 0; i < encodeDatas_.length; i++) {
        this.claim{ value: ethValue_[i] }(encodeDatas_[i]);
    }
}
```
If both `encodeDatas_` and `ethValue_` are empty arrays, the function will not revert. It will accept a transaction with `msg.value` of 0, resulting in a successful transaction that performs no claims.
### Recommendation
- Add a check at the beginning of the `batchClaim` function to ensure that the input arrays are not empty:
```
function batchClaim(bytes[] calldata encodeDatas_, uint256[] calldata ethValue_) external payable {
    if (encodeDatas_.length == 0 || ethValue_.length == 0) revert EmptyArrays();
    if (encodeDatas_.length != ethValue_.length) revert ArrayLengthMismatch();
```

## L-11 Typos/Misspellings
### First instance: Misspelling in `credsMerkeRoot Mapping` (`rood` instead of `root`)
### Impact
The `credsMerkeRoot` mapping in the Cred contract contains a misspelling in its key name. The key is named `rood` instead of `root`, which is likely a typographical error. While this issue may seem minor, it can have several negative impacts:
- Readability
- Consistency
### Proof of concept
The following line in the Cred contract contains the misspelling:
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L45-L46

```solidity
    mapping(uint256 credId => bytes32 rood) public credsMerkeRoot;

```
The key `rood` should be root to accurately reflect its purpose as the `root` of a Merkle tree.
### Recommendation
Correct the misspelling by renaming `rood` to `root`:
```
mapping(uint256 credId => bytes32 root) public credsMerkeRoot
```
### Second instance: Misspelling of `initialized` Variable in `CreatorRoyaltiesControl`
### Impact
The `CreatorRoyaltiesControl.sol` contains a misspelled variable name `initilaized`, which should be `initialized`. This typo can lead to confusion and potential bugs, especially if the variable is referenced elsewhere in the codebase with the correct spelling. Such issues can make the code harder to read, maintain, and debug, potentially leading to logical errors or compilation failures.
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/abstract/CreatorRoyaltiesControl.sol#L13-L14
### Proof of concept
The following code snippet from the CreatorRoyaltiesControl contract demonstrates the issue:
```
bool private initilaized;

function initializeRoyalties(address _royaltyRecipient) internal {
    if (_royaltyRecipient == address(0)) revert InvalidRoyaltyRecipient();
    if (initilaized) revert AlreadyInitialized();  // Typo here
    royaltyRecipient = _royaltyRecipient;
    initilaized = true;  // Typo here
}
```
In this code, the variable `initilaized` is used to track whether the contract has been initialized. However, due to the typo, if someone tries to reference `initialized` (the correct spelling) elsewhere in the code, it would not work as intended.
### Recommendation
The variable name should be corrected from `initilaized` to `initialized`
