## Unused Custom Error.
it is recommended that the definition be removed when custom error is unused.

Found in src/interfaces/IContributeRewards.sol Line: 17

    error SweepFailed();



## The `nonReentrant` `modifier` should occur before all other modifiers.
This is a best-practice to protect against reentrancy in other modifiers.

Found in src/Cred.sol Line: 745

        nonReentrant

## Define and use constant variables instead of using literals

If the same constant literal value is used multiple times, create a constant state variable and reference it throughout the contract.

4 Found Instances
Found in src/PhiFactory.sol Line: 423

        if (protocolFee_ > 10_000) revert ProtocolFeeTooHigh();
Found in src/PhiFactory.sol Line: 431

        if (artCreateFee_ > 10_000) revert ArtCreatFeeTooHigh();
Found in src/abstract/Claimable.sol Line: 35

            msg.data[4:], (bytes32, bytes32, address, address, address, uint256, uint256, uint256, string, bytes32)
Found in src/abstract/Claimable.sol Line: 83

            abi.decode(msg.data[4:], (address, bytes32[], address, uint256, uint256, bytes32, string));





## Missing checks for address(0) when assigning values to address state variables

Several instances found where the address validation is missing. A zero
address validation failure was found when assigning user-supplied address
values to state variables directly.


Found in src/Cred.sol Line: 88
```javascript
        phiSignerAddress = phiSignerAddress_;
```

Found in src/Cred.sol Line: 90
```javascript
        phiRewardsAddress = phiRewardsAddress_;
```

Found in src/Cred.sol Line: 130
```javascript
        phiSignerAddress = phiSignerAddress_;
```

Found in src/Cred.sol Line: 141
```javascript
        protocolFeeDestination = protocolFeeDestination_;
```

Found in src/Cred.sol Line: 155
```javascript
        phiRewardsAddress = phiRewardsAddress_;
```

Found in src/PhiFactory.sol Line: 149
```javascript
        phiSignerAddress = phiSignerAddress_;
```

Found in src/PhiFactory.sol Line: 150
```javascript
        protocolFeeDestination = protocolFeeDestination_;
```

Found in src/PhiFactory.sol Line: 151
```javascript
        erc1155ArtAddress = erc1155ArtAddress_;
```
Found in src/PhiFactory.sol Line: 152
```javascript
        phiRewardsAddress = phiRewardsAddress_;
```

Found in src/PhiFactory.sol Line: 391
```javascript
        phiSignerAddress = phiSignerAddress_;
```

Found in src/PhiFactory.sol Line: 398
```javascript
        phiRewardsAddress = phiRewardsAddress_;
```

Found in src/PhiFactory.sol Line: 405
```javascript
        erc1155ArtAddress = erc1155ArtAddress_;
```
Found in src/PhiFactory.sol Line: 416
```javascript
        protocolFeeDestination = protocolFeeDestination_;
```

Found in src/curve/BondingCurve.sol Line: 35
```javascript
        credContract = ICred(credContract_);
```
Found in src/reward/PhiRewards.sol Line: 69
```javascript
        curatorRewardsDistributor = ICuratorRewardsDistributor(curatorRewardsDistributor_);
```