## Gas Optimization: Avoid Unnecessary Emission of Event in addToWhitelis Function



The addToWhitelis function in the Cred.sol file currently emits an event and sets a value in the curatePriceWhitelist mapping, even if the address is already whitelisted. This unnecessary action consumes extra gas and could be optimized to save approximately 1,500 gas per call.

### Details
 - The function sets the curatePriceWhitelist[address_] mapping to true without checking if the address is already whitelisted.
- It also emits the AddedToWhitelist event every time the function is called, regardless of whether the address was already whitelisted.

  ```solidity
  function addToWhitelist(address address_) external onlyOwner {
    curatePriceWhitelist[address_] = true;
    emit AddedToWhitelist(_msgSender(), address_);
  }


## Gas Optimization Suggestion
- Avoids unnecessary SSTORE operation: Setting a storage value to the same value (true to true) still costs gas in Ethereum.
- Avoids unnecessary event emission: Emitting an event costs additional gas, which can be saved if the event is only emitted when necessary.
- This changes will save around 1500 gas
 ```solidity
function addToWhitelist(address address_) external onlyOwner {
    if (!curatePriceWhitelist[address_]) {
        curatePriceWhitelist[address_] = true;
        emit AddedToWhitelist(_msgSender(), address_);
    }
}