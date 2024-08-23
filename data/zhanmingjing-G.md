https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L171

    function removeFromWhitelist(address address_) external onlyOwner {
        curatePriceWhitelist[address_] = false;
        emit RemovedFromWhitelist(_msgSender(), address_);
    }

should check before set, if curatePriceWhitelist[address_] is not in whitelist, nothing needed to be done.

    function removeFromWhitelist(address address_) external onlyOwner {
        if (curatePriceWhitelist[address_])
        {
            curatePriceWhitelist[address_] = false;
            emit RemovedFromWhitelist(_msgSender(), address_);
        }
    }