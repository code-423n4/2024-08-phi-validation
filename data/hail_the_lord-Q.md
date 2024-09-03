Missing whenNotPaused modifier causes the protocol to not halt and continue the execution of functions which should stop incase of emergency.

## Description
Some contracts have `PausableUpgradeable` contract inherited whose main functionality is to halt the protocol execution completely when in emergency.
But the functions which are responsible to cause state change must have a modifier known as `whenNotPaused` which makes the function useless unless the owner switches them On again.
Below mentioned functions with contract they are in are missing that modifier :
`PhiFactory` :: `updateArtSettings()`, `claim()`, `batchClaim()`
`PhiNFT1155` :: `updateRoyalties()`, `safeTransferFrom()`, `safeBatchTransferFrom`, ``

## Recommended Mitigation Steps
add `whenNotPaused` modifier to the above mentioned functions