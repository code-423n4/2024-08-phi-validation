# Low - 1
Missing whenNotPaused modifier causes the protocol to not halt and continue the execution of functions which should stop incase of emergency.

## Recommended Mitigation Steps
Below mentioned functions with contract they are in are missing `whenNotPaused` modifier :
`PhiFactory` :: `updateArtSettings()`, `claim()`, `batchClaim()`
`PhiNFT1155` :: `updateRoyalties()`, `safeTransferFrom()`, `safeBatchTransferFrom`, ``

# Low - 2
If the artist does not update the art settings using updateArtSettings, they may not receive royalty rewards, missing out on potential earnings.
Which indicates, unless and untill the artist himself updates it, the royalty benefits would be achieved to the protocol.

## Recommended Mitigation Steps
Add the `updateRoyalties()` function, while creating an art using `createArt()` function





