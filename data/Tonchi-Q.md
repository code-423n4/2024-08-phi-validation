### [L-1]function should revert instead of the amount should be equal to balance, accidentally withdraws all money associated

**Description:** In `RewardControl::_withdraw`, if amount == FULL_BALANCE meaning 0, then the function should revert instead of amount should be equal to balance.

**Impact:** when a user accidentally call withdraw with zero than it withdraw all money associated with the address.

**Recommended Mitigation:**
Add a revert if the amount is zero.

### [L-2] Missing checks for amount exceeding address(this).balance

**Description:** In `RewardControl::_withdraw`, This function should have check that amount or balance should not be greater than address(this).balance. 

**Impact:** Call revert if transfer eth more than having in entire contract

**Proof of Concept:** This function doesn't have any checks for amount exceeding contract's balance

**Recommended Mitigation:** Apply checks for same.

### [L-3] Missing checks for artFee exceeding msg.value

**Description:** In `PhiNFT1155::createArtFromFactory` function, Missing checks if artFee is greater than msg.value. This function should have checks for msg.value should always equal or greater than artFee

**Impact:** Transaction revert

**Recommended Mitigation:** Apply this check for msg.value.

### [L-4] Missing Emitting events while changing entire contract

**Description:** In `BondingCurve::setCredContract` function sets address of the credContract but doesn't emit events

**Impact:** No off-chain awareness about this state change can lead some issues. 

**Recommended Mitigation:** Should emit event

### [L-5] supply+amount(targetAmount) greater than TOTAL_SUPPLY_FACTOR gives negative values

**Description:** In `BondingCurve::_curve` function, targetAMount should not be equal or greater than TOTAL_SUPPLY_FACTOR if this happens than it gives supply -ve or undefined value of price

**Impact:** confusion and trasaction revert

**Recommended Mitigation:** Apply checks for targetAmount should always less than TOTAL_SUPPLY_FACTOR

### [L-6] following docs is not prooves right code, code always returns royaltyBPS 500 instead of actual user'ss

**Description:** In `CreatorRoyaltiesControl::getRoyalties` function, following docs is not prooves right code, code
always returns royaltyBPS 500 instead of actual user'ss

**Impact:** 500 is not correct royaltyBPS for every user

### [L-7]following code is dead code, condition never hits

**Description:** In `CreatorRoyaltiesControl::_updateRoyalties`,

```
if (configuration.royaltyRecipient == address(0) && configuration.royaltyBPS > 0) {
            revert InvalidRoyaltyRecipient();
        }
```

**Impact:** Unnecessary bytecode used