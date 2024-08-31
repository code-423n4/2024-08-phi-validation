createCred() allows a sender to create undefined number of Creds while ```expiresIn <= block.timestamp```, as there is no mechanism that prevents from calling createCred() with the same params once and again. The create Creds will have the same properties all of them, which makes little sense for the protocol.

POC

function test_createCred() public {
        vm.warp(START_TIME + 1);

        _createCred("BASIC", "SIGNATURE", 0x0);
    }

function _createCred(string memory credType, string memory verificationType, bytes32 merkleRoot) internal {
        vm.warp(START_TIME + 1);
        vm.startPrank(participant);
        uint256 credId = 1;
        uint256 supply = 0;
        uint256 amount = 1;
        uint256 expiresIn = START_TIME + 100;
        uint256 buyPrice = bondingCurve.getBuyPriceAfterFee(credId, supply, amount);
        string memory credURL = "test";
        bytes memory signCreateData = abi.encode(
            expiresIn, participant, 31_337, address(bondingCurve), credURL, credType, verificationType, merkleRoot
        );
        bytes32 createMsgHash = keccak256(signCreateData);
        bytes32 createDigest = ECDSA.toEthSignedMessageHash(createMsgHash);
        (uint8 cv, bytes32 cr, bytes32 cs) = vm.sign(claimSignerPrivateKey, createDigest);
        if (cv != 27) cs = cs | bytes32(uint256(1) << 255);

        cred.createCred{ value: buyPrice }(participant, signCreateData, abi.encodePacked(cr, cs), 100, 100);
        cred.createCred{ value: buyPrice }(participant, signCreateData, abi.encodePacked(cr, cs), 100, 100);
        cred.createCred{ value: buyPrice }(participant, signCreateData, abi.encodePacked(cr, cs), 100, 100);
        vm.stopPrank();
        assertEq(cred.credIdCounter(),4);
    }

Suggested solution is to add a ```mapping(bytes32 => bool)``` that is set to true when function is succesfully called with a given ```keccak256(signedData)``` and createCred() will have a check that reverts if the ```mapping[keccak256(signedData)]``` is true.