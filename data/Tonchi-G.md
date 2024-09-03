### [G] Missing checks for for amounts[i] > 0 or emit event inside loop

**Description:** In `RewardControl::depositBatch` function, it uses a for loop and doesn't check weather amount is greater than zero or not, it directly add amount to balance of that recipients or should not emit event inside a loop. 

**Impact:** Unnecessary state change or emitting events costs excessive gas.

**Recommended Mitigation:** Apply checks for amounts should greater than zero or emit event outside loop