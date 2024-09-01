
| |Issue|Instances|
|-|:-|:-:|
| GAS-1 | Use require to not set values when it's duplicate | 4 |
### [GAS-1] Use require to not set values when it's duplicate 
Using require to make sure that values are only updated if they are unique that what is already stored. This will reduce the cost from ~2300 gas to ~500 gas in situations when new value is duplicate.

*Instances (4)*:
```solidity
File: /src/art/PhiNFT1155.sol

292:        minterData[tokenId_][to_] = data_;

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#292)

```solidity
File: /src/art/PhiNFT1155.sol

293:        advancedTokenURI[tokenId_][to_] = imageURI_;

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#293)

```solidity
File: /src/Cred.sol

164:        curatePriceWhitelist[address_] = true;

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L164)

```solidity
File: /src/Cred.sol

171:        curatePriceWhitelist[address_] = false;

```
[Link to code](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L171)