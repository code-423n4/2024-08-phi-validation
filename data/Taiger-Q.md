# 1.protocolFeePercent variable has no size limit

## Impact

The `protocolFeePercent` variable has no reasonable size limit, which may cause the protocol fee to be set to an extremely high value. When the protocol fee is set too high, users will fail to pay too high a fee when performing a buy or sell operation. Although the user's funds will not be locked in the contract if the transaction fails, this will hinder the user from performing normal transactions and reduce the usability of the contract.

## Proof of Concept

Suppose that `protocolFeePercent` is set to an extremely high value in the contract, such as 10000 (100%). When a user attempts to buy or sell `Cred`, the calculated protocol fee will be equal to or close to the entire transaction amount, which may cause the user to be unable to pay enough fees to complete the transaction, resulting in a failed transaction.

In the `getPriceData` method, if `protocolFeePercent` is set to a too high value, `protocolFee` will calculate a very high fee, which may cause the user to be unable to pay enough fees to complete the transaction.

The relevant code is as follows:

```
function _getProtocolFee(uint256 price_) internal view returns (uint256) {
    return price_ * credContract.protocolFeePercent() / RATIO_BASE;
}

function getPriceData(
    uint256 credId_,
    uint256 supply_,
    uint256 amount_,
    bool isSign_
)
    public
    view
    returns (uint256 price, uint256 protocolFee, uint256 creatorFee)
{
    price = isSign_ ? getPrice(supply_, amount_) : getPrice(supply_ - amount_, amount_);
    protocolFee = _getProtocolFee(price);
    // ...
}
```
## Tools Used

VSCode

## Recommended Mitigation Steps

When setting `protocolFeePercent`, a reasonable upper limit should be introduced, such as limiting the maximum value of `protocolFeePercent` to 1000 (i.e. 10%).



# 2.Precision Loss and Rounding Error Risk
### Impact

When handling multiple small transactions, precision loss and rounding errors can accumulate, leading to a final total price that is higher than a single large transaction. This discrepancy may cause users to pay more than expected and provide opportunities for arbitrage, leading to unfair competition in the market and potential financial losses.

### Proof of Concept

In the contract code, the `getPrice` function uses the `_curve` method to calculate transaction prices. When a user performs multiple small transactions, each calculation might suffer from precision loss or rounding errors, causing the final cumulative price to be higher than that of a single large transaction.

For example, suppose the current supply is `500 ether`, and a user intends to purchase `10 ether`.

- If the user buys `10 ether` in one go, the calculated price is `100 ether`.
- If the user buys `1 ether` ten times, the final cumulative price might be `101 ether`.

This discrepancy can be exploited by arbitrageurs who split transactions to gain unfair price advantages.

### Tools Used

VSCode

### Recommended Mitigation Steps

1. Introduce high-precision mathematical libraries to ensure consistent pricing across multiple transactions.
2. Implement strict controls over precision and rounding to minimize the price difference between batch and single transactions.

