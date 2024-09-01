You can rewrite for loop in `RewardControl::depositBatch` as the following:

```diff
    function depositBatch(
        address[] calldata recipients,
        uint256[] calldata amounts,
        bytes4[] calldata reasons,
        string calldata comment
    )
        external
        payable
    {
        uint256 numRecipients = recipients.length;

        if (numRecipients != amounts.length || numRecipients != reasons.length) {
            revert ArrayLengthMismatch();
        }

        uint256 expectedTotalValue;

-       for (uint256 i = 0; i < numRecipients; i++) {
+       for (uint256 i; i < numRecipients;) {
            expectedTotalValue += amounts[i];
+
+            unchecked {
+                ++i;
+            }
        }

        if (msg.value != expectedTotalValue) {
            revert InvalidDeposit();
        }

-       for (uint256 i = 0; i < numRecipients; i++) {
+       for (uint256 i; i < numRecipients;) {
            address recipient = recipients[i];
            uint256 amount = amounts[i];

            if (recipient == address(0)) {
                revert InvalidAddressZero();
            }

            balanceOf[recipient] += amount;

            emit Deposit(msg.sender, recipient, reasons[i], amount, comment);
+
+           unchecked {
+               ++i;
+           }
        }
    }
```

Running forge snapshot you'll be able to see the difference in gas consumption

Original Test `TestRewardControl:testDepositBatch(uint8) (runs: 1000, μ: 2567606, ~: 1236839)` Modified one `TestRewardControl:testDepositBatch(uint8) (runs: 1000, μ: 2338887, ~: 1084788)`
