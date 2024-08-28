Prevent Accidental Addition of Zero Address to Whitelist


In the smart contract at https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L163 , the owner could accidentally add the zero address to the whitelist. This scenario should be controlled by a modifier to prevent it. Otherwise, https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L258, the bondingCurve value may be assigned without a revert, resulting in a false return value.