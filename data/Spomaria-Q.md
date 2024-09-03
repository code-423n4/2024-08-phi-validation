### Summary 
The `BondingCurve::setCredContract` function sets an address as the cred contract address. However, the function does not check for address zero before setting the address and as such, risks setting `address(0)` as the cred contract address.

### Vulnerability Details
The vulnerability lies here
```javascript
function setCredContract(address credContract_) external onlyOwner {
    credContract = ICred(credContract_);
}
```
https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/curve/BondingCurve.sol#L34-L36

### Proof of Concept
   
<details>
<summary>PoC</summary>
Place the following code into `Cred.t.sol`.

```javascript
function test__CanSetZeroAddressAsCredContract() public {
    address addressZero = address(0);

    vm.prank(owner);
    bondingCurve.setCredContract(addressZero);

    assertEq(bondingCurve.getCredContract(), addressZero);
}
```

Now run `forge test --match-test test__CanSetZeroAddressAsCredContract -vvvv`

Output:
```javascript
Ran 1 test for test/Cred.t.sol:TestCred
[PASS] test__CanSetZeroAddressAsCredContract() (gas: 14174)
Traces:
  [15180] TestCred::test__CanSetZeroAddressAsCredContract()
    ├─ [0] VM::prank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [7516] BondingCurve::setCredContract(0x0000000000000000000000000000000000000000)
    │   └─ ← [Stop] 
    ├─ [342] BondingCurve::getCredContract() [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000
    ├─ [0] VM::assertEq(0x0000000000000000000000000000000000000000, 0x0000000000000000000000000000000000000000) [staticcall]
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.89ms (106.24µs CPU time)

Ran 1 test suite in 69.39ms (4.89ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

</details>


### Recommended Mitigation Steps
1. Consider adding an `address(0)` check to the `BondingCurve::setCredContract`

