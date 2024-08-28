## Gas Optimization: Avoid Unnecessary Emission of Event in addToWhitelis and function removeFromWhitelist function



The addToWhitelis removeFromWhitelist functions in the Cred.sol file currently emit an event and set a value in the curatePriceWhitelist mapping, even if the address is already whitelisted or it is already removed. This unnecessary action consumes extra gas and could be optimized to save approximately 1,500 gas per call.

### Details
 - The functions set the curatePriceWhitelist[address_] mapping to true or false without checking if the address is already whitelisted or is removed from the whitelist.
- Every time an event is emitted even if the address is already added or removed from the whitelist. 

  ```solidity
  function addToWhitelist(address address_) external onlyOwner {
    curatePriceWhitelist[address_] = true;
    emit AddedToWhitelist(_msgSender(), address_);
  }
  function removeFromWhitelist(address address_) external onlyOwner {
        curatePriceWhitelist[address_] = false;
        emit RemovedFromWhitelist(_msgSender(), address_);
    }


## Gas Optimization Suggestion
- Avoids unnecessary SSTORE operation: Setting a storage value to the same value (true to true) or (false to false) still costs gas in Ethereum.
- Avoids unnecessary event emission: Emitting an event costs additional gas, which can be saved if the event is only emitted when necessary.
- This changes will save around 1500 gas per function
 ```solidity
function addToWhitelist(address address_) external onlyOwner {
    if (!curatePriceWhitelist[address_]) {
        curatePriceWhitelist[address_] = true;
        emit AddedToWhitelist(_msgSender(), address_);
    }
}
function removeFromWhitelist(address address_) external onlyOwner {
        if(curatePriceWhitelist[address_]){
         curatePriceWhitelist[address_] = false;
         emit RemovedFromWhitelist(_msgSender(), address_);
        }
    }