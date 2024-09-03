### L-01: PhiFactory claim function allows user to accidentally lose funds

Although the `merkeClaim` and `signatureClaim` both support sending a refund back to the caller if too much ether is
sent, the `claim` function does not send a refund. The additional value is now only withdrawable to the
`protocolFeeDestination` using the `withdraw` method.

Example demonstrating the issue can be found here: https://github.com/hgrano/2024-08-phi-audit/blob/fc7d16833cab3de5d7c46b844dc68b3cd5db7c16/test/PhiFactory.t.sol#L304-L324

#### Recommendation

Consider refactoring so that the `PhiFactory` does not make `call`s to itself, and therefore it will be easier to track `msg.value` and how much ether has been spent over the course of the function.

### L-02: PhiFactory claim function allows contract funds to be used instead of the caller's

If ether is accidentally sent to the `PhiFactory` (for example, due to the previous issue), then a user can use the
balance to pay for calling the `claim` function, rather than sending the fee through `msg.value`.

Example demonstrating the issue: https://github.com/hgrano/2024-08-phi-audit/blob/fc7d16833cab3de5d7c46b844dc68b3cd5db7c16/test/PhiFactory.t.sol#L326-L351

#### Recommendation

The same recommendation as for L-01 applies.

### L-03: PhiNFT1155 createArtFromFactory sends refund to PhiFactory not to the user

The `createArtFromFactory` method sends a refund back the `msg.sender` (i.e. the `PhiFactory`) rather than the account/user which is creating the art.

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L152

This means the user may end up losing funds to the `PhiFactory`. These funds are then only withdrawable to the `protocolFeeDestination`.

#### Recommendation

Consider moving the refund logic back to the `PhiFactory`. From the context of the `PhiFactory` the `msg.sender` is the correct address to send
the refund to.

### L-04: protocolFeeDestination is not synchronized between PhiNFT1155 and PhiFactory

https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L43C20-L43C42

The `protocolFeeDestination` is set once in the `initialize` function of the `PhiNFT1155`. However, it is possible for this value
to change on the `PhiFactory` via the `setProtocolFeeDestination` function. This does not align with the way the `artCreateFee` is
synchronized between the two contracts, due to the fact it is fetched every time here https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/art/PhiNFT1155.sol#L142.

#### Recommendation

Pass in the `protocolFeeDestination` as a function parameter, or otherwise this variable can be entirely removed
from the `PhiNFT1155` and the logic for sending funds to the `protocolFeeDestination` can be moved to the `PhiFactory`. The
latter approach may be preferrable as it simplifies the contract design.
