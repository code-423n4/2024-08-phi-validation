Some warnings are given below to highlight some dangers that may be treated as medium-level issues and to make sure 
the code operates as intended by the sponsor.

1. Soul-bounded restriction can be easily bypassed.
    https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L317
    https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L343
    Note that the soul-bounded restriction can be easily bypassed by minting tokens to a contract, which, in turn,
    allows the transfer of ownership in a trustless way. At the time of the minting, such a contract may not be present 
    in the chain yet as there are several ways to know a to-be-deployed contract address without exposing it 
    (e.g. by deployer address and nonce or with the help of the `CREATE2` opcode).
    I suggest requiring the `minter` to prove their possession of the corresponding private key by signing a message with it.
    That proves that `minter` is an EOA and not a contract.
    Please make sure that this behavior is expected.

2. Creating an art changes a Merkle root of the entire cred and affects other arts of the cred, preventing minting.
    https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L617
    Several arts can be associated with a cred. However, there is only one Merkle root slot per cred. During art 
    creation, the Merkle root of the entire cred is updated:
    ```solidity
    function _initializePhiArt(PhiArt storage art, ERC1155Data memory createData_) private {
        (
            uint256 credId,
            address credCreator,
            string memory verificationType,
            uint256 credChainId,
            bytes32 merkleRootHash
        ) = abi.decode(createData_.credData, (uint256, address, string, uint256, bytes32));

        art.credId = credId;
        art.credCreator = credCreator;
        // ...

        credMerkleRoot[credChainId][credId] = merkleRootHash;
    }
    ```
    That makes other arts associated with the cred non-mintable unless the new Merkle root incorporates leaves from the
    previous one.
    Please make sure that this behavior is expected.

3. Removing a bonding curve from the whitelist won't affect existing creds using the curve
    https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L170-L173
    Please note that after a cred creation, no attempt is made to ensure that the curve used by the cred is still
    whitelisted. If a curve is removed from the whitelist because of some adverse effects, some creds may 
    continue to use the curve and the adverse effects will prolong. Please make sure that this behavior is expected.
    Also, note that using several bonding curves simultaneously introduces risks of unfair profiting and even asset
    theft as illustrated in other funding.
    
4. An origin check in `_validateAndUpdateClaimState` can be easily bypassed.
    https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L702-L708
    There is a check in `_validateAndUpdateClaimState`:
    ```solidity
        // Common validations
        if (tx.origin != _msgSender() && msg.sender != art.artAddress && msg.sender != address(this)) {
            revert TxOriginMismatch();
        }
    ```
    The purpose of the check is not clear. Usually a check `require(tx.origin == _msgSender())` is used to ensure that  
    the caller is not a contract to avoid some attack paths.
    However, note that in this case the check may be bypassed by calling `claim` function. The function, in turn,
    calls `merkleClaim` or `signatureClaim` via an external call. That leads to the fact that `msg.sender` is 
    `address(this)` for such calls and `revert` won't happen since the condition `msg.sender != address(this)` is not
    satisfied.
    
5. A minter can block others from minting the same art.
    A minter can mint the whole `art.maxSupply` at once, making other participants unable to mint. That raises questions
    particularly in the Merkle claiming case as there are other minters in the Merkle tree (which, presumably, 
    make their way into the tree to eventually mint) deprived of the mint ability.
    Please make sure that this behavior is expected.
