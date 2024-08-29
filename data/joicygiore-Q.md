# L-01
## Title 
The `BondingCurve::_getCreatorFee()` function may return an incorrect `creatorFee`.
## Links to affected code 
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/curve/BondingCurve.sol#L127-L149
## Impact
The function source code is as follows. The `supply_ == 0` branch will not directly return the `creatorFee` value. Continuing to execute will eventually return an incorrect value.
```js
    /// @dev Returns the creator fee.
    function _getCreatorFee(
        uint256 credId_,
        uint256 supply_,
        uint256 price_,
        bool isSign_
    )
        internal
        view
        returns (uint256 creatorFee)
    {
        if (!credContract.isExist(credId_)) {
            return 0;
        }
@>        if (supply_ == 0) {
@>            creatorFee = 0;
        }


        (uint16 buyShareRoyalty, uint16 sellShareRoyalty) = credContract.getCreatorRoyalty(credId_);


        uint16 royaltyRate = isSign_ ? buyShareRoyalty : sellShareRoyalty;
        creatorFee = (price_ * royaltyRate) / RATIO_BASE;
    }
```
## Recommended Mitigation Steps
```diff
    /// @dev Returns the creator fee.
    function _getCreatorFee(
        uint256 credId_,
        uint256 supply_,
        uint256 price_,
        bool isSign_
    )
        internal
        view
        returns (uint256 creatorFee)
    {
        if (!credContract.isExist(credId_)) {
            return 0;
        }
        if (supply_ == 0) {
-            creatorFee = 0;
+            return 0;
        }


        (uint16 buyShareRoyalty, uint16 sellShareRoyalty) = credContract.getCreatorRoyalty(credId_);


        uint16 royaltyRate = isSign_ ? buyShareRoyalty : sellShareRoyalty;
        creatorFee = (price_ * royaltyRate) / RATIO_BASE;
    }
```