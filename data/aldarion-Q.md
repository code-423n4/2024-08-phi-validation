### 1. Excess payment sent to PhiFactory
Context:
 https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L152

Detail:
`PhiNFT1155.createArtFromFactory()` refund any excess ETH to caller but the caller for this function is `PhiFactory`.This results in any excess ETH being refunded to PhiFactory rather than the user. The ETH then becomes stuck in the PhiFactory contract.

Recommendation:
 Use different logic to refund or enforce that the amount of ETH sent with the call matches the required fee exactly. This prevents any excess ETH from needing to be refunded.