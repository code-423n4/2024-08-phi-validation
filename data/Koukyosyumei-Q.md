# Typo in Bonding Curve and PhiNFT1155 Interfaces

## Error 1:

- Location: src/interfaces/IBondingCurve.sol:36:41

```
  --> src/interfaces/IBondingCurve.sol:36:41
   |
36 |         returns (uint256 price, uint256 protcolFee, uint256 creatorFee);
```

- Description: The word `protcol` is used incorrectly. It should be `protocol`.
- Impact: While this error does not directly impact the functionality of the contract, it could lead to confusion or misunderstandings for developers or users interacting with the interface.

## Error 2:

- Location: src/interfaces/IPhiNFT1155.sol:47:17

```
error: `protoco` should be `protocol`
  --> src/interfaces/IPhiNFT1155.sol:47:17
   |
47 |         address protocoFeeDest
```

- Description: The word `protoco` is used incorrectly. It should be `protocol`.
- Impact: Similar to the first error, this typo could potentially cause confusion or hinder readability.