# AI Arena - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. `FighterFarm::reRoll()` gives you the ability to reroll into a `fighterType` you don't own](#H-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: AI Arena

### Dates: Feb 9th, 2024 - Feb 21st, 2024

[See more contest details here](https://code4rena.com/audits/2024-02-ai-arena)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1

# High Risk Findings

## <a id='H-01'></a>H-01. `FighterFarm::reRoll()` gives you the ability to reroll into a `fighterType` you don't own

# Lines of code

[https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L370-L391](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L370-L391)

# Vulnerability details

In the game, there are currently two main types of NFTs: Champions (the standard class) and Dendroids (more exclusive, rare, and difficult to obtain).

The following solidity struct shows the data structure used to represent a `Fighter`. The last property, `dendroidBool`, is a boolean to represent the type of the current Fighter instance, dendroid or not.

```solidity
/// @notice Struct that defines a Fighter NFT.
struct Fighter {
    uint256 weight;
    uint256 element;
    FighterPhysicalAttributes physicalAttributes;
    uint256 id;
    string modelHash;
    string modelType;
    uint8 generation;
    uint8 iconsType;
    bool dendroidBool;
}
```

The problem is that at various points in the code, a `uint8` property called 'fighterType' is used to represent the same information: a Champion type (0 in this case) or a Dendroid type (1 in this case).

One in particular causes the possibility of using the `FighterFarm::reRoll()` functionality to your advantage.

```solidity
function reRoll(uint8 tokenId, uint8 fighterType) public {
    require(msg.sender == ownerOf(tokenId));
@>  require(numRerolls[tokenId] < maxRerollsAllowed[fighterType]);
    require(_neuronInstance.balanceOf(msg.sender) >= rerollCost, "Not enough NRN for reroll");

    _neuronInstance.approveSpender(msg.sender, rerollCost);
    bool success = _neuronInstance.transferFrom(msg.sender, treasuryAddress, rerollCost);
    if (success) {
        numRerolls[tokenId] += 1;
        uint256 dna = uint256(keccak256(abi.encode(msg.sender, tokenId, numRerolls[tokenId])));
@>      (uint256 element, uint256 weight, uint256 newDna) = _createFighterBase(dna, fighterType);
        fighters[tokenId].element = element;
        fighters[tokenId].weight = weight;
@>      fighters[tokenId].physicalAttributes = _aiArenaHelperInstance.createPhysicalAttributes(
            newDna, generation[fighterType], fighters[tokenId].iconsType, fighters[tokenId].dendroidBool
        );
        _tokenURIs[tokenId] = "";
    }
}
```

As you can see in the above code, there is no check about the ownership of the specific `fighterType` you pass as input but only on the `tokenId`.
The `fighterType` is used:

- To check that you have not exceeded your maximum reroll allowance (but comparing the `tokenId` with possibly a `fighterType` you do not own)
- To create the `FighterBase` on which the new dna is based

```solidity
function _createFighterBase(uint256 dna, uint8 fighterType) private view returns (uint256, uint256, uint256) {
@>  uint256 element = dna % numElements[generation[fighterType]];
    uint256 weight = dna % 31 + 65;
@>  uint256 newDna = fighterType == 0 ? dna : uint256(fighterType);
    return (element, weight, newDna);
}
```

- To create the `PhysicalAttributes` of the `Fighter`, using the information of the `generation` for the specific `fighterType`, despite in this case using `fighters[tokenId].dendroidBool` to obtain the Fighter type

The result is that you will have the dna of a dendroid with all its perks but still have a fighter of type 0 (Champion) because `fighters[tokenId].dendroidBool` will still be false.

## Impact

The resulting impact is that if you have, for example, three type 0 fighters, you could re-roll all of them into type 1 fighters with the same cost and without changing the type, only the dna.
The advantages of having a dendroid are purely cosmetic (at the moment), as said in the docs:

> Dendroids are a more exclusive class of NFTs. They have the ability to incorporate other metaverse assets (e.g. NFTs from other projects) into the Arena! These are very rare and more difficult to obtain. You’ll have to follow the story to find out how to get access to one of these!
> It is important to note that Dendroids do not have any performance or attribute advantages over AR-X Bots. The difference is purely cosmetic.

but anyway, not negligible.

## Proof of Concept

Add the following test to `FighterFarm.t.sol`:

```solidity
function testRerollFighter0IntoFighter1() public {
    // I mint the token 0, and I fund myself with some Neurons
    _mintFromMergingPool(_ownerAddress);
    _fundUserWith4kNeuronByTreasury(_ownerAddress);
    if (_fighterFarmContract.ownerOf(0) == _ownerAddress) {
        uint256 neuronBalanceBeforeReRoll = _neuronContract.balanceOf(_ownerAddress);
        uint8 tokenId = 0;

        (,,,,,,,, bool dendroidBool) = _fighterFarmContract.fighters(tokenId);
        // I am the owner of a Champion
        assertFalse(dendroidBool);

        // I reroll into a fighter of type 1
        uint8 fighterType = 1;
        _neuronContract.addSpender(address(_fighterFarmContract));
        _fighterFarmContract.reRoll(tokenId, fighterType);

        (,,,,,,,, dendroidBool) = _fighterFarmContract.fighters(tokenId);
        // Now I still have a Champion but I have the dna of a Dendroid
        assertFalse(dendroidBool);

        assertEq(_fighterFarmContract.numRerolls(0), 1);
        assertEq(neuronBalanceBeforeReRoll > _neuronContract.balanceOf(_ownerAddress), true);
    }
}
```

## Tools Used

Manual review.

## Recommended Mitigation Steps

There are, in my opinion, two options:

1. You could check that the `fighterType` you are passing as input is the same as the token you own and that you are rerolling.

```diff
function reRoll(uint8 tokenId, uint8 fighterType) public {
    require(msg.sender == ownerOf(tokenId));
+   bool isDendroid = fighters[tokenId].dendroidBool;
+   require(
        (isDendroid && fighterType == 1) || (!isDendroid && fighterType == 0),
        "Fighter type mismatch"
    );
    require(numRerolls[tokenId] < maxRerollsAllowed[fighterType]);
    require(_neuronInstance.balanceOf(msg.sender) >= rerollCost, "Not enough NRN for reroll");
    ...
}
```

2. `fighterType` shouldn't be an input of the `reRoll()` function, but it should be calculated based on `dendroidBool`.

```diff
- function reRoll(uint8 tokenId, uint8 fighterType) public {
+ function reRoll(uint8 tokenId) public {
    require(msg.sender == ownerOf(tokenId));
+   bool isDendroid = fighters[tokenId].dendroidBool;
+   uint8 fighterType = isDendroid ? 1 : 0;
    require(numRerolls[tokenId] < maxRerollsAllowed[fighterType]);
    require(_neuronInstance.balanceOf(msg.sender) >= rerollCost, "Not enough NRN for reroll");

    _neuronInstance.approveSpender(msg.sender, rerollCost);
    bool success = _neuronInstance.transferFrom(msg.sender, treasuryAddress, rerollCost);
    if (success) {
        numRerolls[tokenId] += 1;
        uint256 dna = uint256(keccak256(abi.encode(msg.sender, tokenId, numRerolls[tokenId])));
        (uint256 element, uint256 weight, uint256 newDna) = _createFighterBase(dna, fighterType);
        fighters[tokenId].element = element;
        fighters[tokenId].weight = weight;
-       fighters[tokenId].physicalAttributes = _aiArenaHelperInstance.createPhysicalAttributes(
                newDna, generation[fighterType], fighters[tokenId].iconsType, fighters[tokenId].dendroidBool
            );
+       fighters[tokenId].physicalAttributes = _aiArenaHelperInstance.createPhysicalAttributes(
                newDna, generation[fighterType], fighters[tokenId].iconsType, isDendroid
            );
        _tokenURIs[tokenId] = "";
    }
}
```

Personally, I prefer the second option.
Another thing is that it would make the code more readable and easier to use an enum for saving the `fighterType` and `dendroidBool` information. With the Champion as the first value, hence the default.

```solidity
enum Status {
    CHAMPION,
    DENDROID
}
```
