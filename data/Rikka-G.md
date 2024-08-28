Don't Initialize Variables with Default Value

Uninitialized variables are assigned with the types default value.
Explicitly initializing a variable with it's default value costs unnecessary gas.

Instances (3):

File: src/Cred.sol

508: uint256 index = 0;
918: uint256 index = 0;

Link to code: 
https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L508
https://github.com/code-423n4/2024-08-phi/blob/main/src/Cred.sol#L918

Bad:
uint256 index = 0;  // Initialization with default value which is unnecessary

Good: 
uint256 index; // No need to initialize with default value

File: src/reward/CuratorRewardsDistributor.sol

105: uint256 actualDistributeAmount = 0;

Link to code: 
https://github.com/code-423n4/2024-08-phi/blob/main/src/reward/CuratorRewardsDistributor.sol#L105

Bad:
uint256 actualDistributeAmount = 0; // Initialization with default value which is unnecessary

Good: 
uint256 actualDistributeAmount; // No need to initialize with default value

Description:

Bad: uint256 index = 0; and uint256 actualDistributeAmount = 0; are redundant because uint256 variables are automatically initialized to 0 if no explicit value is provided.

Good: uint256 index; and uint256 actualDistributeAmount; leverage Solidity's default behavior, avoiding unnecessary gas consumption and making the code cleaner.
