## [H-01] `FighterFarm.mintFromMergingPool()` create fighter with not random stats

### Relevant GitHub Links

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L307-L331

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L458-L474

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L476-L531

## Summary

`FighterFarm.mintFromMergingPool` should create a random `FighterOps.Fighter` instance and mint it to user when this function is called by `MergingPool` contract, but in fact, anyone can pre-compute `FighterOps.Fighter` before function is called. Result of execution `FighterFarm.mintFromMergingPool` is not random because it relies on mathematical operations and storage data which can be seen and manipulated by any user.

## Vulnerability Details

Function `FighterFarm.mintFromMergingPool`, does not operate with random data to create random fighter stats.

```solidity
    function mintFromMergingPool(
        address to,
        string calldata modelHash,
        string calldata modelType,
        uint256[2] calldata customAttributes
    ) public {
        require(msg.sender == _mergingPoolAddress);
        _createNewFighter(
            to,
@>          uint256(keccak256(abi.encode(msg.sender, fighters.length))),
            modelHash,
            modelType,
            0,
            0,
            customAttributes
        );
    }
```

Instead, it uses expression `uint256(keccak256(abi.encode(msg.sender, fighters.length)))` which operates with contract's storage and incoming executor address which are not random. So any user can execute this operation with same data by executing `FighterFarm.mintFromMergingPool` function and get the same result.

## Impact

1. Attacker can get better `FighterOps.Fighter` by pre-computing result of `FighterFarm.mintFromMergingPool`, execute function only when computing return fighter that have necessary stats. Thus attacker will have better fighters than other users.
2. 1. Attacker see some user's pending transaction of `FighterFarm.mintFromMergingPool`.
   2. Attacker pre-computes result of this operation and see user will get `FighterOps.Fighter` with good stats.
   3. To prevent getting good fighter by user, attacker frontruns this operation by calling any of functions that change dependent data (`FighterFarm.claimFighters()`, `FighterFarm.redeemMintPass()`, `FighterFarm.reRoll()`) and user won't get that fighter.

## Proof of Concept

1. For testing purposes add function `getFightersLength` to `FighterFarm` contract to get length of the array of fighters.

```solidity
  /// @notice Get length of the array of fighters.
  /// @return Length of the array of fighters.
  function getFightersLength() public view returns (uint256) {
      return fighters.length;
  }
```

2. Add this test to `tests/FighterFarm.t.sol` and run with `forge test --match-contract FighterFarm --match-test testMintFromMergingPoolCreateNotRandomFighter`

```solidity
  function _createFighterBase(
      uint256 dna,
      uint8 fighterType
  ) private view returns (uint256, uint256, uint256) {
        // Get number of elements per generation
      uint8 numElementsPerGeneration = _fighterFarmContract.numElements(
          fighterType
      );

      uint256 element = dna % numElementsPerGeneration;
      uint256 weight = (dna % 31) + 65;
      uint256 newDna = fighterType == 0 ? dna : uint256(fighterType);
      return (element, weight, newDna);
  }

  function _createNewFighter(
      uint256 dna,
      string memory modelHash,
      string memory modelType,
      uint8 fighterType,
      uint8 iconsType,
      uint256[2] memory customAttributes
  ) private view returns (FighterOps.Fighter memory) {
      // Get generation of fighter by it's type
      uint8 generationPerFighterType = _fighterFarmContract.generation(
          fighterType
      );

      uint256 element;
      uint256 weight;
      uint256 newDna;
      if (customAttributes[0] == 100) {
          (element, weight, newDna) = _createFighterBase(dna, fighterType);
      } else {
          element = customAttributes[0];
          weight = customAttributes[1];
          newDna = dna;
      }

      // Get length of fighters array
      uint256 newId = _fighterFarmContract.getFightersLength();

      bool dendroidBool = fighterType == 1;
      FighterOps.FighterPhysicalAttributes memory attrs = _helperContract
          .createPhysicalAttributes(
              newDna,
              generationPerFighterType,
              iconsType,
              dendroidBool
          );

      return
          FighterOps.Fighter(
              weight,
              element,
              attrs,
              newId,
              modelHash,
              modelType,
              generationPerFighterType,
              iconsType,
              dendroidBool
          );
  }

  function testMintFromMergingPoolCreateNotRandomFighter() public {
      // Get length of fighters array
      uint256 fightersLength = _fighterFarmContract.getFightersLength();

      // Make same actions as in mintFromMergingPool with data from _mintFromMergingPool function from this test
      FighterOps.Fighter memory expectedFighter = _createNewFighter(
          uint256(
              keccak256(
                  abi.encode(address(_mergingPoolContract), fightersLength)
              )
          ),
          "_neuralNetHash",
          "original",
          0,
          0,
          [uint256(1), uint256(80)]
      );

      // Actual mint of new fighter
      _mintFromMergingPool(_ownerAddress);

      // Created fighter data
      (
          uint256 weight,
          uint256 element,
          FighterOps.FighterPhysicalAttributes memory physicalAttributes,
          uint256 id,
          string memory modelHash,
          string memory modelType,
          uint8 generation,
          uint8 iconsType,
          bool dendroidBool
      ) = _fighterFarmContract.fighters(0);

      // Check whether expected fighter data is the same as actual
      assertEq(expectedFighter.weight, weight);
      assertEq(expectedFighter.element, element);
      assertEq(
          expectedFighter.physicalAttributes.head,
          physicalAttributes.head
      );
      assertEq(
          expectedFighter.physicalAttributes.eyes,
          physicalAttributes.eyes
      );
      assertEq(
          expectedFighter.physicalAttributes.mouth,
          physicalAttributes.mouth
      );
      assertEq(
          expectedFighter.physicalAttributes.body,
          physicalAttributes.body
      );
      assertEq(
          expectedFighter.physicalAttributes.hands,
          physicalAttributes.hands
      );
      assertEq(
          expectedFighter.physicalAttributes.feet,
          physicalAttributes.feet
      );
      assertEq(expectedFighter.id, id);
      assertEq(expectedFighter.modelHash, modelHash);
      assertEq(expectedFighter.modelType, modelType);
      assertEq(expectedFighter.generation, generation);
      assertEq(expectedFighter.iconsType, iconsType);
      assertEq(expectedFighter.dendroidBool, dendroidBool);
  }
```

3. Test result:

```bash
  Running 1 test for test/FighterFarm.t.sol:FighterFarmTest
  [PASS] testMintFromMergingPoolCreateNotRandomFighter() (gas: 473493)
  Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.95ms

  Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual Review & Foundry.

## Recommendations

Consider using a decentralized oracle for the generation of random numbers, such as [Chainlink VRF](https://docs.chain.link/vrf).

## [H-02] `FighterFarm.reRoll()` change fighter stats with non-random data

### Relevant GitHub Links

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L367-L391

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L458-L474

## Summary

`FighterFarm.reRoll` should `roll a new fighter with random traits`, but in fact, anyone can pre-compute result of this function before function is called. Result of execution `FighterFarm.reRoll` is not random because it relies on mathematical operations and storage data which can be seen and manipulated by any user.

## Vulnerability Details

Function `FighterFarm.reRoll`, does not operate with random data to recreate random fighter stats.

```solidity
    function reRoll(uint8 tokenId, uint8 fighterType) public {
        require(msg.sender == ownerOf(tokenId));
        require(numRerolls[tokenId] < maxRerollsAllowed[fighterType]);
        require(
            _neuronInstance.balanceOf(msg.sender) >= rerollCost,
            "Not enough NRN for reroll"
        );

        _neuronInstance.approveSpender(msg.sender, rerollCost);
        bool success = _neuronInstance.transferFrom(
            msg.sender,
            treasuryAddress,
            rerollCost
        );
        if (success) {
            numRerolls[tokenId] += 1;
            uint256 dna = uint256(
@>               keccak256(abi.encode(msg.sender, tokenId, numRerolls[tokenId]))
            );
            (
                uint256 element,
                uint256 weight,
                uint256 newDna
            ) = _createFighterBase(dna, fighterType);
```

Instead, it uses expression `uint256(keccak256(abi.encode(msg.sender, tokenId, numRerolls[tokenId])))` which operates with contract's storage and incoming executor address and tokenId which are not random. So any user can execute this operation with same data by executing `FighterFarm.reRoll` function and get the same result.

## Impact

Attacker can get better `FighterOps.Fighter` by pre-computing result of `FighterFarm.reRoll`, execute function only when computing return fighter that have necessary stats. Thus attacker can get better fighters than other users.

## Proof of Concept

1. For testing purposes add function `getFightersLength` to `FighterFarm` contract to get length of the array of fighters.

```solidity
    /// @notice Get length of the array of fighters.
    /// @return Length of the array of fighters.
    function getFightersLength() public view returns (uint256) {
        return fighters.length;
    }
```

2. Add this test to `tests/FighterFarm.t.sol` and run with `forge test --match-contract FighterFarm --match-test testReRollCreateNotRandomFighter`

```solidity
    function _createFighterBase(
        uint256 dna,
        uint8 fighterType
    ) private view returns (uint256, uint256, uint256) {
        // Get number of elements per generation
        uint8 numElementsPerGeneration = _fighterFarmContract.numElements(
            fighterType
        );

        uint256 element = dna % numElementsPerGeneration;
        uint256 weight = (dna % 31) + 65;
        uint256 newDna = fighterType == 0 ? dna : uint256(fighterType);
        return (element, weight, newDna);
    }

    function testReRollCreateNotRandomFighter() public {
        _neuronContract.addSpender(address(_fighterFarmContract));
        _fundUserWith4kNeuronByTreasury(_ownerAddress);

        // Get length of fighters array
        uint256 tokenId = _fighterFarmContract.getFightersLength();
        uint8 fighterType = 0;

        // Mint new fighter
        _mintFromMergingPool(_ownerAddress);

        // Minted fighter data
        (
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            uint8 mintedFighterIconsType,
            bool mintedFighterDendroidBool
        ) = _fighterFarmContract.fighters(tokenId);

        // Compute expected reRoll result
        uint8 numRerollsOfFighter = _fighterFarmContract.numRerolls(tokenId);
        uint8 generationPerFighterType = _fighterFarmContract.generation(
            fighterType
        );
        uint256 dna = uint256(
            keccak256(
                abi.encode(_ownerAddress, tokenId, numRerollsOfFighter + 1)
            )
        );

        (
            uint256 expectedElement,
            uint256 expectedWeight,
            uint256 newDna
        ) = _createFighterBase(dna, fighterType);

        FighterOps.FighterPhysicalAttributes
            memory expectedAttrs = _helperContract.createPhysicalAttributes(
                newDna,
                generationPerFighterType,
                mintedFighterIconsType,
                mintedFighterDendroidBool
            );

        // Execute reRoll
        _fighterFarmContract.reRoll(uint8(tokenId), fighterType);

        // Rerolled fighter data
        (
            uint256 weight,
            uint256 element,
            FighterOps.FighterPhysicalAttributes memory physicalAttributes,
            ,
            ,
            ,
            ,
            ,

        ) = _fighterFarmContract.fighters(tokenId);

        // Check whether expected fighter data is the same as actual
        assertEq(expectedWeight, weight);
        assertEq(expectedElement, element);
        assertEq(expectedAttrs.head, physicalAttributes.head);
        assertEq(expectedAttrs.eyes, physicalAttributes.eyes);
        assertEq(expectedAttrs.mouth, physicalAttributes.mouth);
        assertEq(expectedAttrs.body, physicalAttributes.body);
        assertEq(expectedAttrs.hands, physicalAttributes.hands);
        assertEq(expectedAttrs.feet, physicalAttributes.feet);
    }
```

3. Test result:

```bash
    Running 1 test for test/FighterFarm.t.sol:FighterFarmTest
    [PASS] testReRollCreateNotRandomFighter() (gas: 657233)
    Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 14.20ms

    Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual Review & Foundry.

## Recommendations

Consider using a decentralized oracle for the generation of random numbers, such as [Chainlink VRF](https://docs.chain.link/vrf).

## [H-03] `FighterFarm.claimFighters()` create not random fighters

### Relevant GitHub Links

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L185-L222

## Summary

`FighterFarm.claimFighters` should create a random `FighterOps.Fighter` instances and mint them to user, but in fact, anyone can pre-compute `FighterOps.Fighter` before function is called. Result of execution `FighterFarm.claimFighters` is not random because it relies on mathematical operations and storage data which can be seen and manipulated by any user.

## Vulnerability Details

Function `FighterFarm.claimFighters`, does not operate with random data to create random fighters stats.

```solidity
    function claimFighters(
        uint8[2] calldata numToMint,
        bytes calldata signature,
        string[] calldata modelHashes,
        string[] calldata modelTypes
    )
        external
    {
        bytes32 msgHash = bytes32(keccak256(abi.encode(
            msg.sender,
            numToMint[0],
            numToMint[1],
            nftsClaimed[msg.sender][0],
            nftsClaimed[msg.sender][1]
        )));
        require(Verification.verify(msgHash, signature, _delegatedAddress));
        uint16 totalToMint = uint16(numToMint[0] + numToMint[1]);
        require(modelHashes.length == totalToMint && modelTypes.length == totalToMint);
        nftsClaimed[msg.sender][0] += numToMint[0];
        nftsClaimed[msg.sender][1] += numToMint[1];
        for (uint16 i = 0; i < totalToMint; i++) {
            _createNewFighter(
                msg.sender,
@>              uint256(keccak256(abi.encode(msg.sender, fighters.length))),
                modelHashes[i],
                modelTypes[i],
                i < numToMint[0] ? 0 : 1,
                0,
                [uint256(100), uint256(100)]
            );
        }
    }
```

Instead, it uses expression `uint256(keccak256(abi.encode(msg.sender, fighters.length)))` which operates with contract's storage and incoming executor address which are not random. So any user can execute this operation with same data by executing `FighterFarm.claimFighters` function and get the same result.

## Impact

1. Attacker can get better `FighterOps.Fighter` instances by pre-computing result of `FighterFarm.claimFighters`, execute function only when computing return fighters that have necessary stats. Thus attacker will have better fighters than other users.
2. 1. Attacker see some user's pending transaction of `FighterFarm.claimFighters`.
   2. Attacker pre-computes result of this operation and see user will get `FighterOps.Fighter` instances with good stats.
   3. To prevent getting good fighters by user, attacker frontruns this operation by calling any of functions that change dependent data (`FighterFarm.claimFighters()`, `FighterFarm.redeemMintPass()`, `FighterFarm.reRoll()`) and user won't get good fighters.

## Proof of Concept

1. For testing purposes add function `getFightersLength` to `FighterFarm` contract to get length of the array of fighters.

```solidity
    /// @notice Get length of the array of fighters.
    /// @return Length of the array of fighters.
    function getFightersLength() public view returns (uint256) {
        return fighters.length;
    }
```

2. Add this test to `tests/FighterFarm.t.sol` and run with `forge test --match-contract FighterFarm --match-test testClaimFightersCreateNotRandomFighter`

```solidity
    function _createFighterBase(
        uint256 dna,
        uint8 fighterType
    ) private view returns (uint256, uint256, uint256) {
        // Get number of elements per generation
        uint8 numElementsPerGeneration = _fighterFarmContract.numElements(
            fighterType
        );

        uint256 element = dna % numElementsPerGeneration;
        uint256 weight = (dna % 31) + 65;
        uint256 newDna = fighterType == 0 ? dna : uint256(fighterType);
        return (element, weight, newDna);
    }

    function _createNewFighter(
        uint256 dna,
        string memory modelHash,
        string memory modelType,
        uint8 fighterType,
        uint8 iconsType,
        uint256[2] memory customAttributes
    ) private view returns (FighterOps.Fighter memory) {
        // Get generation of fighter by it's type
        uint8 generationPerFighterType = _fighterFarmContract.generation(
            fighterType
        );

        uint256 element;
        uint256 weight;
        uint256 newDna;
        if (customAttributes[0] == 100) {
            (element, weight, newDna) = _createFighterBase(dna, fighterType);
        } else {
            element = customAttributes[0];
            weight = customAttributes[1];
            newDna = dna;
        }

        // Get length of fighters array
        uint256 newId = _fighterFarmContract.getFightersLength();

        bool dendroidBool = fighterType == 1;
        FighterOps.FighterPhysicalAttributes memory attrs = _helperContract
            .createPhysicalAttributes(
                newDna,
                generationPerFighterType,
                iconsType,
                dendroidBool
            );

        return
            FighterOps.Fighter(
                weight,
                element,
                attrs,
                newId,
                modelHash,
                modelType,
                generationPerFighterType,
                iconsType,
                dendroidBool
            );
    }

    function testClaimFightersCreateNotRandomFighter() public {
        uint8[2] memory numToMint = [1, 0];
        bytes memory claimSignature = abi.encodePacked(
            hex"407c44926b6805cf9755a88022102a9cb21cde80a210bc3ad1db2880f6ea16fa4e1363e7817d5d87e4e64ba29d59aedfb64524620e2180f41ff82ca9edf942d01c"
        );
        string[] memory claimModelHashes = new string[](1);
        claimModelHashes[
            0
        ] = "ipfs://bafybeiaatcgqvzvz3wrjiqmz2ivcu2c5sqxgipv5w2hzy4pdlw7hfox42m";

        string[] memory claimModelTypes = new string[](1);
        claimModelTypes[0] = "original";

        // Get length of fighters array
        uint256 fightersLength = _fighterFarmContract.getFightersLength();

        FighterOps.Fighter memory expectedFighter = _createNewFighter(
            uint256(keccak256(abi.encode(_ownerAddress, fightersLength))),
            claimModelHashes[0],
            claimModelTypes[0],
            0,
            0,
            [uint256(100), uint256(100)]
        );

        // Claim fighter
        _fighterFarmContract.claimFighters(
            numToMint,
            claimSignature,
            claimModelHashes,
            claimModelTypes
        );

        // Created fighter data
        (
            uint256 weight,
            uint256 element,
            FighterOps.FighterPhysicalAttributes memory physicalAttributes,
            uint256 id,
            string memory modelHash,
            string memory modelType,
            uint8 generation,
            uint8 iconsType,
            bool dendroidBool
        ) = _fighterFarmContract.fighters(fightersLength);

        // Check whether expected fighter data is the same as actual
        assertEq(expectedFighter.weight, weight);
        assertEq(expectedFighter.element, element);
        assertEq(
            expectedFighter.physicalAttributes.head,
            physicalAttributes.head
        );
        assertEq(
            expectedFighter.physicalAttributes.eyes,
            physicalAttributes.eyes
        );
        assertEq(
            expectedFighter.physicalAttributes.mouth,
            physicalAttributes.mouth
        );
        assertEq(
            expectedFighter.physicalAttributes.body,
            physicalAttributes.body
        );
        assertEq(
            expectedFighter.physicalAttributes.hands,
            physicalAttributes.hands
        );
        assertEq(
            expectedFighter.physicalAttributes.feet,
            physicalAttributes.feet
        );
        assertEq(expectedFighter.id, id);
        assertEq(expectedFighter.modelHash, modelHash);
        assertEq(expectedFighter.modelType, modelType);
        assertEq(expectedFighter.generation, generation);
        assertEq(expectedFighter.iconsType, iconsType);
        assertEq(expectedFighter.dendroidBool, dendroidBool);
    }
```

3. Test result:

```bash
    Running 1 test for test/FighterFarm.t.sol:FighterFarmTest
    [PASS] testClaimFightersCreateNotRandomFighter() (gas: 581714)
    Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.90ms

    Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual Review & Foundry.

## Recommendations

Consider using a decentralized oracle for the generation of random numbers, such as [Chainlink VRF](https://docs.chain.link/vrf).

## [H-04] `FighterFarm.redeemMintPass()` create not random fighters

### Relevant GitHub Links

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L224-L262

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L458-L474

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L476-L531

## Summary

`FighterFarm.redeemMintPass` should create a random `FighterOps.Fighter` instances and mint them to user, but in fact, anyone can pre-compute `FighterOps.Fighter` before function is called. Result of execution `FighterFarm.redeemMintPass` is not random because it relies on mathematical operations and storage data which can be seen and manipulated by any user.

## Vulnerability Details

Function `FighterFarm.redeemMintPass`, does not operate with random data to create random fighters stats.

```solidity
    function redeemMintPass(
        uint256[] calldata mintpassIdsToBurn,
        uint8[] calldata fighterTypes,
        uint8[] calldata iconsTypes,
        string[] calldata mintPassDnas,
        string[] calldata modelHashes,
        string[] calldata modelTypes
    )
        external
    {
        require(
            mintpassIdsToBurn.length == mintPassDnas.length &&
            mintPassDnas.length == fighterTypes.length &&
            fighterTypes.length == modelHashes.length &&
            modelHashes.length == modelTypes.length
        );
        for (uint16 i = 0; i < mintpassIdsToBurn.length; i++) {
            require(msg.sender == _mintpassInstance.ownerOf(mintpassIdsToBurn[i]));
            _mintpassInstance.burn(mintpassIdsToBurn[i]);
            _createNewFighter(
                msg.sender,
@>              uint256(keccak256(abi.encode(mintPassDnas[i]))),
                modelHashes[i],
                modelTypes[i],
                fighterTypes[i],
                iconsTypes[i],
                [uint256(100), uint256(100)]
            );
        }
    }
```

Instead, it uses expression `uint256(keccak256(abi.encode(mintPassDnas[i])))` which operates with contract's storage and incoming mint pass DNA which are not random. So any user can execute this operation with the same data by executing `FighterFarm.redeemMintPass` function and get the same result.

## Impact

1. Attacker can get better `FighterOps.Fighter` instances by pre-computing result of `FighterFarm.redeemMintPass`, execute function only when computing return fighters that have necessary stats. Thus attacker will have better fighters than other users.
2. 1. Attacker see some user's pending transaction of `FighterFarm.redeemMintPass`.
   2. Attacker pre-computes result of this operation and see user will get `FighterOps.Fighter` instances with good stats.
   3. To prevent getting good fighters by user, attacker frontruns this operation by calling any of functions that change dependent data (`FighterFarm.claimFighters()`, `FighterFarm.redeemMintPass()`, `FighterFarm.reRoll()`) and user won't get good fighters.

## Proof of Concept

1. For testing purposes add function `getFightersLength` to `FighterFarm` contract to get length of the array of fighters.

```solidity
    /// @notice Get length of the array of fighters.
    /// @return Length of the array of fighters.
    function getFightersLength() public view returns (uint256) {
        return fighters.length;
    }
```

2. Add this test to `tests/FighterFarm.t.sol` and run with `forge test --match-contract FighterFarm --match-test testRedeemMintPassCreateNotRandomFighter`

```solidity
    function _createFighterBase(
        uint256 dna,
        uint8 fighterType
    ) private view returns (uint256, uint256, uint256) {
        // Get number of elements per generation
        uint8 numElementsPerGeneration = _fighterFarmContract.numElements(
            fighterType
        );

        uint256 element = dna % numElementsPerGeneration;
        uint256 weight = (dna % 31) + 65;
        uint256 newDna = fighterType == 0 ? dna : uint256(fighterType);
        return (element, weight, newDna);
    }

    function _createNewFighter(
        uint256 dna,
        string memory modelHash,
        string memory modelType,
        uint8 fighterType,
        uint8 iconsType,
        uint256[2] memory customAttributes
    ) private view returns (FighterOps.Fighter memory) {
        // Get generation of fighter by it's type
        uint8 generationPerFighterType = _fighterFarmContract.generation(
            fighterType
        );

        uint256 element;
        uint256 weight;
        uint256 newDna;
        if (customAttributes[0] == 100) {
            (element, weight, newDna) = _createFighterBase(dna, fighterType);
        } else {
            element = customAttributes[0];
            weight = customAttributes[1];
            newDna = dna;
        }

        // Get length of fighters array
        uint256 newId = _fighterFarmContract.getFightersLength();

        bool dendroidBool = fighterType == 1;
        FighterOps.FighterPhysicalAttributes memory attrs = _helperContract
            .createPhysicalAttributes(
                newDna,
                generationPerFighterType,
                iconsType,
                dendroidBool
            );

        return
            FighterOps.Fighter(
                weight,
                element,
                attrs,
                newId,
                modelHash,
                modelType,
                generationPerFighterType,
                iconsType,
                dendroidBool
            );
    }

    function testRedeemMintPassCreateNotRandomFighter() public {
        uint8[2] memory numToMint = [1, 0];
        bytes memory signature = abi.encodePacked(
            hex"20d5c3e5c6b1457ee95bb5ba0cbf35d70789bad27d94902c67ec738d18f665d84e316edf9b23c154054c7824bba508230449ee98970d7c8b25cc07f3918369481c"
        );
        string[] memory _tokenURIs = new string[](1);
        _tokenURIs[
            0
        ] = "ipfs://bafybeiaatcgqvzvz3wrjiqmz2ivcu2c5sqxgipv5w2hzy4pdlw7hfox42m";

        _mintPassContract.claimMintPass(numToMint, signature, _tokenURIs);

        uint256[] memory _mintpassIdsToBurn = new uint256[](1);
        string[] memory _mintPassDNAs = new string[](1);
        uint8[] memory _fighterTypes = new uint8[](1);
        uint8[] memory _iconsTypes = new uint8[](1);
        string[] memory _neuralNetHashes = new string[](1);
        string[] memory _modelTypes = new string[](1);

        _mintpassIdsToBurn[0] = 1;
        _mintPassDNAs[0] = "dna";
        _fighterTypes[0] = 0;
        _neuralNetHashes[0] = "neuralnethash";
        _modelTypes[0] = "original";
        _iconsTypes[0] = 1;

        _mintPassContract.approve(address(_fighterFarmContract), 1);

        // Get length of fighters array
        uint256 fightersLength = _fighterFarmContract.getFightersLength();

        FighterOps.Fighter memory expectedFighter = _createNewFighter(
            uint256(keccak256(abi.encode(_mintPassDNAs[0]))),
            _neuralNetHashes[0],
            _modelTypes[0],
            _fighterTypes[0],
            _iconsTypes[0],
            [uint256(100), uint256(100)]
        );

        _fighterFarmContract.redeemMintPass(
            _mintpassIdsToBurn,
            _fighterTypes,
            _iconsTypes,
            _mintPassDNAs,
            _neuralNetHashes,
            _modelTypes
        );

        (
            uint256 weight,
            uint256 element,
            FighterOps.FighterPhysicalAttributes memory physicalAttributes,
            uint256 id,
            string memory modelHash,
            string memory modelType,
            uint8 generation,
            uint8 iconsType,
            bool dendroidBool
        ) = _fighterFarmContract.fighters(fightersLength);

        assertEq(expectedFighter.weight, weight);
        assertEq(expectedFighter.element, element);
        assertEq(
            expectedFighter.physicalAttributes.head,
            physicalAttributes.head
        );
        assertEq(
            expectedFighter.physicalAttributes.eyes,
            physicalAttributes.eyes
        );
        assertEq(
            expectedFighter.physicalAttributes.mouth,
            physicalAttributes.mouth
        );
        assertEq(
            expectedFighter.physicalAttributes.body,
            physicalAttributes.body
        );
        assertEq(
            expectedFighter.physicalAttributes.hands,
            physicalAttributes.hands
        );
        assertEq(
            expectedFighter.physicalAttributes.feet,
            physicalAttributes.feet
        );
        assertEq(expectedFighter.id, id);
        assertEq(expectedFighter.modelHash, modelHash);
        assertEq(expectedFighter.modelType, modelType);
        assertEq(expectedFighter.generation, generation);
        assertEq(expectedFighter.iconsType, iconsType);
        assertEq(expectedFighter.dendroidBool, dendroidBool);
    }
```

3. Test result:

```bash
    Running 1 test for test/FighterFarm.t.sol:FighterFarmTest
    [PASS] testRedeemMintPassCreateNotRandomFighter() (gas: 686783)
    Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.13ms

    Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual Review & Foundry.

## Recommendations

Consider using a decentralized oracle for the generation of random numbers, such as [Chainlink VRF](https://docs.chain.link/vrf).

## [M-01] Non-transferable game item can be transfered with `safeBatchTransferFrom`

### Relevant GitHub Links

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/GameItems.sol#L289-L303

## Summary

When admin creates a new game item with `transferable == false`, this game item is considered as non-transferable so any account that has this game item should not transfer it to another account. `GameItem.safeTransferFrom` have require statement to check whether game item is transferable or not, but `ERC1155.sol` have another function to transfer tokens - `safeBatchTransferFrom`, which does not have the same require statement. In that case, user can transfer game item with `ERC1155.safeBatchTransferFrom`.

## Impact

Users can transfer non-transferable game item with `safeBatchTransferFrom`.

## Proof of Concept

1. Add this test to `test/GameItems.t.sol` and run it with `forge test --match-contract GameItems --match-test testSafeBatchTransferFromCanTransferNonTransferableGameItem`.

```solidity
  function testSafeBatchTransferFromCanTransferNonTransferableGameItem()
      public
  {
      address from = makeAddr("from");
      address to = makeAddr("to");

      uint256 tokenId = 1;
      uint256 amount = 1;

      // create new non-transferable game item
      _gameItemsContract.createGameItem(
          "Non Transferable Game Item",
          "https://ipfs.io/ipfs/",
          true,
          false,
          4,
          1 * 10 ** 18,
          1
      );

      _fundUserWith4kNeuronByTreasury(from);

      // mint 1 token to "from" address
      vm.startPrank(from);
      _gameItemsContract.mint(1, amount);

      // revert if we use saveTransferFrom
      vm.expectRevert();
      _gameItemsContract.safeTransferFrom(from, to, tokenId, amount, "");

      uint256[] memory tokenIds = new uint256[](1);
      uint256[] memory amounts = new uint256[](1);

      tokenIds[0] = 1;
      amounts[0] = 1;

      // but safeBatchTransferFrom will successfully transfer game item that should not be transferable
      _gameItemsContract.safeBatchTransferFrom(
          from,
          to,
          tokenIds,
          amounts,
          ""
      );

      vm.stopPrank();
  }
```

2. Test result:

```bash
  Running 1 test for test/GameItems.t.sol:GameItemsTest
  [PASS] testSafeBatchTransferFromCanTransferNonTransferableGameItem() (gas: 291315)
  Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.12ms

  Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual Review & Foundry.

## Recommended Mitigation Steps

For changing transfer functionality consider using `ERC1155._beforeTokenTransfer`.

```diff
-    /// @notice Safely transfers an NFT from one address to another.
-    /// @dev Added a check to see if the game item is transferable.
-    function safeTransferFrom(
-        address from,
-        address to,
-        uint256 tokenId,
-        uint256 amount,
-        bytes memory data
-    ) public override(ERC1155) {
-        require(allGameItemAttributes[tokenId].transferable);
-        super.safeTransferFrom(from, to, tokenId, amount, data);
-    }
+    /// @dev Check if tokens can be transfered in before token transfer hook
+    function _beforeTokenTransfer(
+        address /*operator */,
+        address from,
+        address to,
+        uint256[] memory ids,
+        uint256[] memory /* amounts */,
+        bytes memory /* data */
+    ) internal virtual override {
+        if (from != address(0) && to != address(0)) {
+            uint256 idsLength = ids.length;
+
+            for (uint256 i = 0; i < idsLength; i++) {
+                require(allGameItemAttributes[ids[i]].transferable);
+            }
+        }
+    }
```

## [M-02] `MergingPool.getFighterPoints` out of bound if maxId more then 1

### Relevant GitHub Links

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/MergingPool.sol#L202-L212

## Summary

If user pass `maxId` more then 1 in `MergingPool.getFighterPoints` it will revert because it create fixed array with size of 1. So setting value at index more then 0 resulting function to revert.

## Impact

User can not get fighters points if fighter have index more then 0 in `fighterPoints` mapping.

## Proof of Concept

Add this test to `tests/MergingPool.t.sol` and run with `forge test --match-contract MergingPool --match-test test_RevertIfMaxIdIsMoreThenOneInGetFighterPoints`. It should revert.

```solidity
  function test_RevertIfMaxIdIsMoreThenOneInGetFighterPoints() public {
      vm.expectRevert();
      // this will revert
      _mergingPoolContract.getFighterPoints(3);
  }
```

Result:

```bash
  Running 1 test for test/MergingPool.t.sol:MergingPoolTest
  [PASS] test_RevertIfMaxIdIsMoreThenOneInGetFighterPoints() (gas: 15140)
  Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.65ms

  Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual Review & Foundry.

## Recommended Mitigation Steps

Make fixed array with length of maxId.

```diff
  function getFighterPoints(
        uint256 maxId
    ) public view returns (uint256[] memory) {
-     uint256[] memory points = new uint256[](1);
+     uint256[] memory points = new uint256[](maxId);
      for (uint256 i = 0; i < maxId; i++) {
          points[i] = fighterPoints[i];
      }
      return points;
  }
```

## [M-03] `FighterFarm` does not properly implement ERC165

### Relevant GitHub Links

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L406-L417

## Summary

The [`EIP-165`](https://eips.ethereum.org/EIPS/eip-165): Standard Interface Detection implementation is incomplete, missing the implementation of function supportsInterface(bytes4 interfaceId) external view returns (bool).

## Impact

Harder to identify by other contracts which interfaces or functionalities `FighterFarm` supports.

## Tools Used

Manual Review & Foundry.

## Recommended Mitigation Steps

Add interface id of `FighterFarm` to `FighterFarm.supportsInterface`.

```diff
  function supportsInterface(
      bytes4 _interfaceId
  ) public view override(ERC721, ERC721Enumerable) returns (bool) {
-     return super.supportsInterface(_interfaceId);
+     return _interfaceId == 0xcf3eb250 || super.supportsInterface(_interfaceId);
  }
```

## [G-01] Remove unnecessary storage read from `VoltageManager.useVoltageBattery` when emitting event `VoltageRemaining`

### Relevant GitHub Links

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/VoltageManager.sol#L91-L99

## Summary

Function `VoltageManager.useVoltageBattery` replenishes voltage for a player, and emits an event that represents the remaining amount of voltage that player has. As this function sets remaining voltage of player to `100` there is no need to read `ownerVoltage[msg.sender]` when event emits.

## Recommended Mitigation Steps

Change second argument in `VoltageRemaining` in `VoltageManager.useVoltageBattery` from `ownerVoltage[msg.sender]` to `100`.

```diff
  function useVoltageBattery() public {
      require(ownerVoltage[msg.sender] < 100);
      require(_gameItemsContractInstance.balanceOf(msg.sender, 0) > 0);
      _gameItemsContractInstance.burn(msg.sender, 0, 1);
      ownerVoltage[msg.sender] = 100;
-     emit VoltageRemaining(msg.sender, ownerVoltage[msg.sender]);
+     emit VoltageRemaining(msg.sender, 100);
  }
```

# [QA-01] Unused event `TokensMinted` in `Neuron` contract

## Relevant GitHub Links

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/Neuron.sol#L20-L21

## Summary

Event `TokensMinted` is not used but by NatSpec it should be used in `Neuron.mint` function.

## Recommended Mitigation Steps

Add event in `Neuron.mint`.

```diff
  function mint(address to, uint256 amount) public virtual {
      require(totalSupply() + amount < MAX_SUPPLY, "Trying to mint more than the max supply");
      require(hasRole(MINTER_ROLE, msg.sender), "ERC20: must have minter role to mint");
      _mint(to, amount);
+     emit TokensMinted(to, amount);
  }
```

# [QA-02] No need to inherit `ERC721` by `FighterFarm`

## Relevant GitHub Links

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L16

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L333-L348

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L350-L365

## Summary

There is no need to inherit `ERC721` by `FighterFarm` because `ERC721Enumerable` already inherits `ERC721`.

## Recommended Mitigation Steps

1. Remove inheritance of `ERC721` by `FighterFarm`:

```diff
-   contract FighterFarm is ERC721, ERC721Enumerable {
+   contract FighterFarm is ERC721Enumerable {
```

2. Fix override statements in functions `FighterFarm.tokenURI`, `FighterFarm.supportsInterface`:

```diff
-   function tokenURI(uint256 tokenId) public view override(ERC721) returns (string memory) {
+   function tokenURI(uint256 tokenId) public view override returns (string memory) {
        return _tokenURIs[tokenId];
    }
```

```diff
    function supportsInterface(bytes4 _interfaceId)
        public
        view
-       override(ERC721, ERC721Enumerable)
+       override
        returns (bool)
    {
        return super.supportsInterface(_interfaceId);
    }
```

3. Remove redundant empty `FighterFarm._beforeTokenTransfer` function:

```diff
-   /*//////////////////////////////////////////////////////////////
-                           INTERNAL FUNCTIONS
-   //////////////////////////////////////////////////////////////*/
-
-   /// @notice Hook that is called before a token transfer.
-   /// @param from The address transferring the token.
-   /// @param to The address receiving the token.
-   /// @param tokenId The ID of the NFT being transferred.
-   function _beforeTokenTransfer(address from, address to, uint256 tokenId)
-       internal
-       override(ERC721, ERC721Enumerable)
-   {
-       super._beforeTokenTransfer(from, to, tokenId);
-   }
```
