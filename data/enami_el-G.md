### G-01 Gas Inefficiency in `addToWhitelist` Function Due to Unnecessary State Changes

### Description:
The `addToWhitelist` function sets the `address_` in `curatePriceWhitelist` to `true` without checking if it is already `true`. This can lead to unnecessary state changes on the blockchain, consuming additional gas. If the address is already whitelisted, updating the state again is redundant and results in wasted gas.

### Part of Code:
```solidity
function addToWhitelist(address address_) external onlyOwner {
    curatePriceWhitelist[address_] = true;
    emit AddedToWhitelist(_msgSender(), address_);
}
```
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L163C1-L166C6
### Recommendation:
To optimize gas usage, include a condition to check whether the address is already whitelisted before updating the state. The modified code could look like this:

```solidity
function addToWhitelist(address address_) external onlyOwner {
    if (!curatePriceWhitelist[address_]) {
        curatePriceWhitelist[address_] = true;
        emit AddedToWhitelist(_msgSender(), address_);
    }
}
```

### G-02 Gas Inefficiency in `removeFromWhitelist` Function Due to Unnecessary State Changes

### Description:
The `removeFromWhitelist` function sets the `address_` in `curatePriceWhitelist` to `false` without checking if it is already `false`. This can lead to unnecessary state changes on the blockchain, resulting in higher gas consumption. If the address is already removed from the whitelist, updating the state again is redundant and leads to wasted gas.

### Part of Code:
```solidity
function removeFromWhitelist(address address_) external onlyOwner {
    curatePriceWhitelist[address_] = false;
    emit RemovedFromWhitelist(_msgSender(), address_);
}
```
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L170C2-L173C6
### Recommendation:
Include a condition to check whether the address is already removed from the whitelist before updating the state. The modified code could look like this:

```solidity
function removeFromWhitelist(address address_) external onlyOwner {
    if (curatePriceWhitelist[address_]) {
        curatePriceWhitelist[address_] = false;
        emit RemovedFromWhitelist(_msgSender(), address_);
    }
}
```

