### cred::_validateAndCalculateBatch()
`priceLimits_` array should also be validated for length of elements in the array to match the length. Else there is a potential for revert due to out of bounds exception.

```
+  if (length != priceLimits_.length) {
+            revert InvalidArrayLength();
+  }
``` 

### cred::getBatchBuyPrice()
`credIds_` array and `amounts_`  array lengths should be validated to be request to avoid out of bounds exception and reverts.

### cred::getBatchSellPrice()
`credIds_` array and `amounts_`  array lengths should be validated to be request to avoid out of bounds exception and reverts.

