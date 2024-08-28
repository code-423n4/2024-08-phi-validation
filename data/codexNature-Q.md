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