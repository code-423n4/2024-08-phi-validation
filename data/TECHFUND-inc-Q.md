### cred::_validateAndCalculateBatch()
`priceLimits_` array should also be validated for length of elements in the array to match the length. Else there is a potential for revert due to out of bounds exception.

```
+  if (length != priceLimits_.length) {
+            revert InvalidArrayLength();
+  }
``` 