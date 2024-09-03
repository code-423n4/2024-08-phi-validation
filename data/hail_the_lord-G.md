# Gas - 1
`credContract` must be `immutable` in `CuratorRewardsDistributor`
`RATIO_BASE` must be `constant` isntead of `immutable` in `CuratorRewardsDistributor`
`MAX_ROYALTY_RANGE` must be `constant` isntead of `immutable` in `CuratorRewardsDistributor`


# Gas - 2
artChainId attribute is not used anywhere in the protocol, but still exist for every art created.

## Description
The attribute `artChainId` exist for every art created, but is not used anywhere, leading to excess gas while creating an art.

## Recommended Mitigation Steps
Remove it from `_createERC1155Data` functions
