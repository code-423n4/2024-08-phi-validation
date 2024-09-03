# Team Polarized Light - Phi Protocol - QA Report

## [L-01] Incorrect Error Handling for Maximum Supply in BondingCurve Contract

### Overview

The BondingCurve contract does not properly handle the scenario when the maximum supply is reached, leading to unexpected errors and potential vulnerabilities.

### Description

The contract's `_curve` function, which is crucial for price calculations, will revert with a division by zero error when the target amount reaches the maximum supply (999). This occurs because the function attempts to divide by zero when `TOTAL_SUPPLY_FACTOR - targetAmount_` becomes zero. Additionally, for amounts exceeding 1000, the function will revert due to an underflow in the same calculation.

### Code Location

The issue is in the `_curve` function of the `BondingCurve` contract:

```solidity
function _curve(uint256 targetAmount_) private pure returns (uint256) {
    return (TOTAL_SUPPLY_FACTOR * CURVE_FACTOR * 1 ether) / (TOTAL_SUPPLY_FACTOR - targetAmount_)
        - CURVE_FACTOR * 1 ether - INITIAL_PRICE_FACTOR * targetAmount_ / 1000;
}
```

### Impact

This issue can lead to unexpected transaction failures and confusing error messages when the system approaches or exceeds its maximum capacity. It may also create vulnerabilities that could be exploited to cause denial of service or other unintended behaviors.

### Recommended Mitigations

1. Implement a check in the `_curve` function to handle the case when `targetAmount_` is equal to or greater than `TOTAL_SUPPLY_FACTOR`:

```solidity
function _curve(uint256 targetAmount_) private pure returns (uint256) {
    if (targetAmount_ >= TOTAL_SUPPLY_FACTOR) {
        revert MaxSupplyReached();
    }
    return (TOTAL_SUPPLY_FACTOR * CURVE_FACTOR * 1 ether) / (TOTAL_SUPPLY_FACTOR - targetAmount_)
        - CURVE_FACTOR * 1 ether - INITIAL_PRICE_FACTOR * targetAmount_ / 1000;
}
```

2. Add appropriate error handling in public functions that call `_curve`, such as `getPrice`, to catch and properly handle the `MaxSupplyReached` error.

3. Consider implementing a custom error for maximum supply reached scenarios:

```solidity
error MaxSupplyReached();
```

4. Update the `getPrice` function to handle the maximum supply case:

```solidity
function getPrice(uint256 supply_, uint256 amount_) public pure returns (uint256) {
    if (supply_ + amount_ > 999) {
        revert MaxSupplyReached();
    }
    return _curve((supply_ + amount_) * 1 ether) - _curve(supply_ * 1 ether);
}
```

## [L-02] Contracts with multiple onlyXYZ modifiers where XYZ is a role can introduce complexities when managing privileges

### Overview

The smart contracts, particularly `PhiNFT1155.sol`, employ multiple role-based access control modifiers (e.g., `onlyOwner`, `onlyAdmin`, etc.). While this approach provides granular control, it can lead to complications in privilege management and reduce overall contract maintainability.

### Description

The use of multiple `onlyXYZ` modifiers for different roles, while offering fine-grained access control, can make it challenging to manage and audit permissions effectively. This approach can lead to a complex web of permissions that becomes difficult to maintain, especially as the contract grows or undergoes updates. It may also increase the risk of accidentally granting excessive privileges or overlooking necessary restrictions.

### Code Location

This issue is primarily observed in `PhiNFT1155.sol`:

https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L21-L85

### Impact

The complexity introduced by this approach can lead to several issues:
1. Increased difficulty in auditing and understanding the permission structure
2. Higher risk of misconfiguration, potentially granting unintended access
3. Challenges in maintaining and updating the contract as requirements evolve
4. Potential for overlooking necessary access controls on critical functions

### Recommended Mitigations

To address this issue, consider implementing a more robust and flexible role-based access control system. Specifically:

1. Utilize OpenZeppelin's AccessControl contract, which provides a standardized and well-tested approach to role-based access control.
2. Define clear roles with specific sets of permissions, rather than creating individual modifiers for each privileged function.

By adopting a more structured approach to access control, the contract can become more maintainable, easier to audit, and less prone to permission-related vulnerabilities.

## [L-03] State variables not capped at reasonable values

### Overview

Certain state variables within the smart contracts, particularly in `Cred.sol`, lack defined boundaries or caps on their possible values. This absence of limits could potentially lead to unexpected behavior or vulnerabilities if these variables reach extreme values.

### Description

Setting appropriate boundaries on state variables is crucial for maintaining system integrity and protecting users from potential exploits or unintended consequences. Without caps on values, variables could theoretically reach extremes that either exploit contract functionality or disrupt normal operations. This is particularly concerning for variables that directly impact financial calculations or control critical contract parameters.

### Code Location

`setProtocolFeePercent` function within `Cred.sol`. 
 
https://github.com/code-423n4/2024-08-phi/blob/main/src/abstract/RewardControl.sol#L122-L126
 
### Impact

The lack of bounds on these variables could lead to several potential issues:
1. Extreme values could cause unexpected behavior in calculations, potentially leading to financial losses.
2. Malicious actors with the ability to modify these variables could set them to values that effectively break contract functionality.
3. Accidental misconfigurations could go undetected, leading to long-term issues in contract operation.
4. Users might lose trust in the protocol if they observe unreasonable values for critical parameters.

### Recommended Mitigations

1. Introduce minimum and maximum permissible values for all critical state variables.
2. In setter functions like `setProtocolFeePercent`, add require statements to enforce these boundaries.
3. Consider using OpenZeppelin's SafeMath library for arithmetic operations involving these variables to prevent overflow and underflow.
4. Implement events that are emitted when these variables are changed, allowing for off-chain monitoring of critical parameter updates.

Example implementation for `setProtocolFeePercent`:

```solidity
uint256 public constant MAX_PROTOCOL_FEE_PERCENT = 1000; // 10% as the maximum fee
uint256 public constant MIN_PROTOCOL_FEE_PERCENT = 0;

function setProtocolFeePercent(uint256 _newFeePercent) external onlyOwner {
    require(_newFeePercent >= MIN_PROTOCOL_FEE_PERCENT && _newFeePercent <= MAX_PROTOCOL_FEE_PERCENT, "Fee out of bounds");
    protocolFeePercent = _newFeePercent;
    emit ProtocolFeePercentUpdated(_newFeePercent);
}
```

## [L-04] Missing events in functions that are either setters, privileged or voting related

### Overview

Several critical functions lack event emissions, particularly in setter functions, privileged operations, and voting-related actions.

### Description

Event emissions play a crucial role in smart contract design, serving as an on-chain logging mechanism that allows external observers and dapps to track and react to important state changes. 

### Code Location

Main examples of this are present in the following:

https://github.com/code-423n4/2024-08-phi/blob/main/src/curve/BondingCurve.sol#L34-L34
https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L195-L195
https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L156-L156
https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L161-L161
https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L170-L170
https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L176-L176

### Impact

The lack of event emissions in these functions can lead to several issues:
1. Reduced transparency: External observers cannot easily track important state changes.
2. Difficulty in creating reactive systems: dApps and other smart contracts that depend on these contracts may struggle to respond to state changes in real-time.
3. User experience degradation: Users may not receive immediate feedback about the success or failure of their actions.

### Recommended Mitigations

1. Identify all setter functions, privileged operations, and voting-related actions across the contract suite.
2. For each identified function, define and emit an appropriate event that captures the relevant details of the state change.
3. Ensure that events include all pertinent information, such as the address initiating the change, old and new values (for setters), and any other relevant parameters.

Example implementation for a setter function:

```solidity
event ProtocolFeeUpdated(address indexed updater, uint256 oldFee, uint256 newFee);

function setProtocolFee(uint256 _newFee) external onlyOwner {
    uint256 oldFee = protocolFee;
    protocolFee = _newFee;
    emit ProtocolFeeUpdated(msg.sender, oldFee, _newFee);
}
```

## [L-05] Upgradeable contracts should have a __gap variable

### Overview

The upgradeable contracts within the system, specifically `Cred.sol`, `PhiFactory.sol`, and `PhiNFT1155.sol`, lack a `__gap` variable. This omission could potentially lead to storage layout incompatibilities in future upgrades.

### Description

In upgradeable contracts, the `__gap` variable serves as a component for maintaining storage layout compatibility across different versions. It reserves storage slots that can be used to add new state variables in future upgrades without affecting the storage layout of existing variables. Without this gap, adding new state variables in upgraded versions could potentially overwrite or corrupt existing data.

### Code Location

This issue affects the following upgradeable contracts:
- `Cred.sol` - https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L19-L19
- `PhiFactory.sol` - https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L22-L22
- `PhiNFT1155.sol` - https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L21-L21

### Impact

The absence of a `__gap` variable in these upgradeable contracts could lead to several potential issues:
1. Difficulty in adding new state variables: Future upgrades that require new state variables may be impossible without breaking storage compatibility.
2. Risk of data corruption: If upgrades are attempted without proper precautions, existing data could be overwritten or corrupted.
3. Limited flexibility: The lack of reserved storage slots reduces the contract's ability to evolve over time.
4. Increased upgrade complexity: Each upgrade would require careful manual management of the storage layout to avoid conflicts.

### Recommended Mitigations

1. Add a `__gap` variable to each upgradeable contract. This should be a uint256 array with a size that leaves ample room for future additions.
2. Ensure that the `__gap` variable is placed at the end of the storage layout in each contract.

Example implementation:

```solidity
contract UpgradeableContract is Initializable {
    // Existing state variables...

    // Reserved storage space to allow for new state variables in future upgrades.
    // The size of this array should be calculated carefully based on the contract's needs.
    uint256[50] private __gap;
}
```

## [L-06] SafeTransferLib does not ensure that the token contract exists

### Overview

The use of SafeTransferLib for token transfers in the smart contracts, particularly in `PhiFactory.sol`, introduces a potential vulnerability due to the library's lack of contract existence checks.

### Description

SafeTransferLib, while providing some safety checks for token transfers, does not verify whether the token contract actually exists at the specified address. This oversight can lead to silent failures in token transfer operations, as the library will not revert if it attempts to interact with a non-existent contract.

### Code Location

`PhiFactory.sol` - https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L17-L17

### Impact

The lack of contract existence checks can lead to several potential issues:
1. Silent failures: Token transfers may fail without reverting, leading to a false sense of success.
2. Loss of funds: In extreme cases, tokens could be sent to non-existent addresses, effectively burning them.
3. Inconsistent state: The contract's state may become inconsistent if it assumes a transfer was successful when it actually failed.
4. Difficulty in debugging: Silent failures are notoriously hard to debug and may lead to complex issues down the line.

### Recommended Mitigations

1. Perform explicit contract existence checks before using SafeTransferLib functions.
2. Implement a wrapper function that combines contract existence checks with SafeTransferLib operations.
3. Add events that emit the result of token transfer operations to aid in debugging and monitoring.


## [L-07] Inconsistent expiry logic using block global values

### Overview

The smart contracts, particularly `PhiFactory.sol`, exhibit inconsistent use of comparison operators when checking against `block.timestamp` for expiry logic. This inconsistency can lead to ambiguity and potential off-by-one errors in timing-critical operations.

### Description

The contract uses a mix of less-than (`<`) and less-than-or-equal-to (`<=`) operators when comparing current time (`block.timestamp`) against expiry timestamps. This lack of standardization can lead to subtle bugs and unexpected behavior, especially in edge cases where precise timing is crucial.

### Code Location

`PhiFactory.sol` - https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L22-L236

Line 236: `if (endTime_ < block.timestamp) {`
Line 339: `if (expiresIn_ <= block.timestamp) revert SignatureExpired();`
Line 585: `if (createData_.endTime <= block.timestamp) revert EndTimeInPast();`
Line 592: `if (expiresIn_ <= block.timestamp) revert SignatureExpired();`

### Impact

The inconsistent use of comparison operators can lead to multiple issues:

1. Off-by-one errors: Functions may execute (or fail to execute) at unexpected times due to inconsistent comparisons.
2. Ambiguity in contract behavior: Users and developers may misunderstand when certain actions are valid or expired.
3. Potential for exploitation: In extreme cases, attackers could exploit these inconsistencies to perform actions at unintended times.

### Recommended Mitigations

1. Review all time-based logic in the contract suite to ensure consistency.
2. Add clear comments explaining the exact timing behavior intended for each comparison.

## [L-08] Inconsistent use of _msgSender() and msg.sender in contract
### Overview

The smart contracts exhibit inconsistent usage of `_msgSender()` and `msg.sender` for identifying the caller of a function. This inconsistency can lead to unexpected behavior, especially in more complex contract interactions or when using meta-transactions.

### Description

`_msgSender()` is typically used in contracts that support meta-transactions or in upgradeable contracts to allow for future flexibility in sender identification. Using `msg.sender` directly can limit this flexibility and lead to inconsistencies in how the contract identifies the transaction initiator.

### Code Location

This inconsistency is observed across multiple contracts, including:

- `PhiFactory.sol`
  - Line 112 `if (arts[artId_].artist != _msgSender()) revert NotArtCreator();`
  - Line 706 `if (tx.origin != _msgSender() && msg.sender != art.artAddress && msg.sender != address(this)) {`
  - Line 740 `_msgSender().safeTransferETH(etherValue_ - mintFee);`

- `PhiNFT1155.sol` - https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L21-L120
  - Line 80 `if (msg.sender != artist && msg.sender != owner()) revert NotArtCreator();`
  - Line 86 `if (msg.sender != address(phiFactoryContract)) revert NotPhiFactory();`
  - Line 105 `__Ownable_init(msg.sender);`
  - Line 120 `phiFactoryContract = IPhiFactory(payable(msg.sender));`
  - Line 152 `_msgSender().safeTransferETH(msg.value - artFee);`
  - Line 318 `address sender = _msgSender();`
  - Line 345 `address sender = _msgSender();`

- `CuratorRewardsDistributor.sol` - https://github.com/code-423n4/2024-08-phi/blob/main/src/reward/CuratorRewardsDistributor.sol#L17-L37
  - Line 37 - `constructor(address phiRewardsContract_, address credContract_) payable Ownable(msg.sender)`
  - Line 120 `_msgSender().safeTransferETH(royaltyfee + distributeAmount - actualDistributeAmount);`
  - Line 128 `credId, _msgSender(), royaltyfee + distributeAmount - actualDistributeAmount, distributeAmount, totalBalance`

### Impact

The inconsistent use of `_msgSender()` and `msg.sender` can lead to issues including:

1. Incompatibility with meta-transactions: Parts of the contract may not work correctly if meta-transactions are implemented in the future.
2. Inconsistent behavior: Different functions may behave differently with regard to sender identification, leading to confusion and potential vulnerabilities.
3. Difficulty in upgrading: If the contract needs to be upgraded to support new transaction types, the inconsistent use of sender identification methods can complicate the process.

### Recommended Mitigations

1. Standardize the use of `_msgSender()` throughout the contract suite for identifying the transaction initiator.
2. Review all instances of `msg.sender` and replace them with `_msgSender()` unless there's a specific reason to use `msg.sender` directly.
3. Add clear comments explaining any cases where `msg.sender` is intentionally used instead of `_msgSender()`.

## [L-09] Deposit/redeem functions vulnerable to donation attack

### Overview

The `_withdraw` function in the RewardControl contract implements accounting arithmetic that is vulnerable to the donation attack.

### Description

The `_withdraw` function allows for a full balance withdrawal using a special `FULL_BALANCE` constant. This implementation can potentially be exploited in a donation attack scenario, where an attacker could manipulate balances by donating tokens.

### Code Location

File: `src/abstract/RewardControl.sol`

```solidity
function _withdraw(address from, address to, uint256 amount) internal {
    if (to == address(0)) revert InvalidAddressZero();

    uint256 balance = balanceOf[from];
    if (amount == FULL_BALANCE) {
        amount = balance;
    }

    if (amount > balance) revert InvalidAmount();

    unchecked {
        balanceOf[from] = balance - amount;
    }

    emit Withdraw(from, to, amount);

    to.safeTransferETH(amount);
}
```

### Impact

This vulnerability could potentially be exploited to manipulate balances, leading to unexpected behavior or financial losses.

### Recommended Mitigations

1. Implement a check-effects-interactions pattern to prevent potential reentrancy.
2. Consider using OpenZeppelin's `SafeERC20` library for token transfers to add an extra layer of security.
3. Implement a reentrancy guard modifier on functions that interact with external contracts.

## [NC-01] Sweeping may break accounting if tokens with multiple addresses are used

### Overview

The contract's sweeping functionality does not account for the possibility of tokens with multiple controlling addresses. This oversight could potentially lead to accounting errors or unexpected behavior when dealing with such tokens.

### Description

Historically, there have been instances where tokens inadvertently had two controlling addresses, such that transfers made via one address would affect the balance of the other. The current sweeping mechanism in the contract does not consider this edge case, which could lead to accounting discrepancies.

### Code Location

`PhiFactory.sol` https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L579-L580

### Impact

While this issue is classified as non-critical, it could potentially lead to the following problems:
1. Accounting errors: The contract's internal accounting may become inaccurate if it interacts with a token that has multiple controlling addresses.
2. Unexpected behavior: Sweeping operations might not work as intended for tokens with this unusual property.
3. Potential for exploitation: In extreme cases, this could be exploited to manipulate the contract's understanding of token balances.

### Recommended Mitigations

1. Implement a validation step after sweeping operations to ensure that the balance of non-sweepable or exempt tokens remains unaffected.
2. Add a mechanism to manually reconcile token balances in case discrepancies are detected.
3. Consider maintaining a whitelist of trusted tokens that are known to behave correctly.
4. Implement events that log the before and after balances of swept tokens to aid in auditing and detecting discrepancies.

Example implementation of a post-sweep validation:

```solidity
function sweepTokens(address token) external {
    uint256 preSweepBalance = IERC20(token).balanceOf(address(this));
    
    // Perform sweeping operation
    // ...

    uint256 postSweepBalance = IERC20(token).balanceOf(address(this));
    require(postSweepBalance == expectedPostSweepBalance, "Unexpected balance change");
    
    emit TokensSwept(token, preSweepBalance - postSweepBalance);
}

event TokensSwept(address indexed token, uint256 amount);
```

## [NC-02] An event should be emitted if a non-immutable state variable is set in a constructor

### Overview

The `CuratorRewardsDistributor.sol` contract sets non-immutable state variables in its constructor without emitting corresponding events. This practice reduces transparency and makes it more difficult for off-chain systems to track the contract's initial state.

### Description

When important state variables are set in a constructor, especially those that are non-immutable, it's a best practice to emit events. This allows external systems to easily track the initial configuration of the contract and any subsequent changes to these variables.

### Code Location

`CuratorRewardsDistributor.sol`.
https://github.com/code-423n4/2024-08-phi/blob/main/src/reward/CuratorRewardsDistributor.sol#L37-L42 

### Impact

While not a critical issue, it can lead to the following:

1. Reduced transparency: External observers cannot easily track the initial state of important contract variables.
2. Difficulty in auditing: It becomes more challenging to verify the correct initialization of the contract.
3. Inconsistent practice: Not emitting events for important state changes in the constructor is inconsistent with best practices for other state-changing functions.

### Recommended Mitigations

1. Identify all non-immutable state variables set in the constructor.
2. Define events for each of these state variables.
3. Emit these events at the end of the constructor after all state variables have been set.
4. Consider creating a single "Initialized" event that captures all initial state, if there are many variables.

Example implementation:

```solidity
contract CuratorRewardsDistributor {
    address public curator;
    uint256 public rewardRate;

    event CuratorSet(address indexed newCurator);
    event RewardRateSet(uint256 newRewardRate);

    constructor(address _curator, uint256 _rewardRate) {
        curator = _curator;
        rewardRate = _rewardRate;

        emit CuratorSet(_curator);
        emit RewardRateSet(_rewardRate);
    }

    // Rest of the contract...
}
```

## [NC-03] Inconsistent checks of address parameters against address(0)

### Overview

There are inconsistencies in how address parameters are checked against the zero address across different functions in the contracts.

### Description

Some functions perform zero address checks on their parameters, while others do not. This inconsistency could lead to potential issues if zero addresses are accidentally passed to functions without checks.

### Code Location

Examples found in:
- `src/Cred.sol`
- https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L69-L72
- https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L544-L549

- `src/art/PhiNFT1155.sol`
- https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L307-L309
- https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L332-L334

### Impact

While not immediately critical, this inconsistency could lead to unexpected behavior or make the code more prone to errors in the future.

### Recommended Mitigations

1. Implement consistent zero address checks for all address parameters across all functions.
2. Consider creating a reusable modifier or internal function for zero address checks to ensure consistency.

## [NC-04] Variables declared with names of defined functions

### Overview

Some variables in the contracts are declared with names that match defined functions within the project.

### Description

Using variable names that are the same as function names can lead to confusion and make the code less readable. It may also cause issues with name shadowing in certain contexts.

### Code Location

Examples found in:
- `src/PhiFactory.sol` 
- https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L282-L282
- https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L738-L739

- `src/art/PhiNFT1155.sol`
- https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L58-L58

### Impact

While not a security risk, this practice reduces code clarity and could lead to maintenance issues or bugs in future development.

### Recommended Mitigations

1. Rename variables to avoid conflicts with function names.
2. Establish and follow a consistent naming convention for variables and functions to prevent future conflicts.

## [NC-05] Integers passed into abi.encodePacked without casting to string

### Overview

In some instances, integers are passed directly into `abi.encodePacked` without first being cast to strings.

### Description

Not casting integers to strings before passing them into `abi.encodePacked` can result in unintended encoding behavior. For example, the integer 1 might be encoded as the character with ASCII value 1, rather than the string "1".

### Code Location

Examples found in:
- `src/PhiFactory.sol` - https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L620-L631
- `src/art/PhiNFT1155.sol` - https://github.com/code-423n4/2024-08-phi/blob/main/src/art/PhiNFT1155.sol#L96-L119

### Impact

This could lead to unexpected results when decoding the packed data, potentially causing issues in functions that rely on this encoded data.

### Recommended Mitigations

1. Cast all integers to strings before passing them to `abi.encodePacked`.
2. Use a library like OpenZeppelin's `Strings` for consistent and safe integer-to-string conversion.

## [NC-06] For loops iterate on arrays without explicit length checks

### Overview

Several for loops in the contracts iterate over arrays without first explicitly checking that the lengths of all relevant arrays match.

### Description

When multiple arrays are used in a single loop, it's important to ensure that all arrays have the expected length. Failure to do so could result in out-of-bounds access or incomplete processing of data.

### Code Location

Examples found in:

- `src/PhiFactory.sol` 
- https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L313-L314 ,
- https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L526-L528


- `src/reward/CuratorRewardsDistributor.sol` 
- https://github.com/code-423n4/2024-08-phi/blob/main/src/reward/CuratorRewardsDistributor.sol#L37-L42 

- `src/Cred.sol`
- https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L751-L753
- https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L772-L781

### Impact

This could potentially lead to logical errors, incomplete operations, or in worst-case scenarios, out-of-bounds access causing runtime errors.

### Recommended Mitigations

1. Add explicit length checks before loops that operate on multiple arrays.
2. Consider using a custom error to revert if array lengths don't match, providing clear feedback on the issue.

## [NC-07] Single modifier checks msg.sender against multiple addresses

### Overview

In the `PhiNFT1155` contract, a single modifier checks `msg.sender` against two different addresses.

### Description

The `onlyArtCreator` modifier checks if `msg.sender` is either the artist or the contract owner. While functional, this approach reduces the granularity of access control and may make the contract less flexible for future modifications.

### Code Location

File: `src/art/PhiNFT1155.sol`

```solidity
modifier onlyArtCreator(uint256 tokenId_) {
    uint256 artId = _tokenIdToArtId[tokenId_];
    address artist = phiFactoryContract.artData(artId).artist;
    if (msg.sender != artist && msg.sender != owner()) revert NotArtCreator();
    _;
}
```

### Impact

This design choice may make it more difficult to adjust access control in the future and could potentially lead to overly permissive access in some scenarios.

### Recommended Mitigations

1. Split the modifier into two separate modifiers: one for the artist and one for the owner.
2. Use OpenZeppelin's `AccessControl` contract for more granular and extensible role-based access control.
