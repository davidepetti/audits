# First Flight #10: One Shot - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `RapBattle::battle()` uses a weak RNG](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Reentrancy vulnerability in `OneShot::mintRapper()` allow users to temporarily gain additional skills, thereby increasing their chances of winning battles during that period](#M-01)
- ## Low Risk Findings
    - ### [L-01. `RapBattle::_battle()` emits incorrect data on `Battle` event](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #10

### Dates: Feb 22nd, 2024 - Feb 29th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-02-one-shot)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. `RapBattle::battle()` uses a weak RNG            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L62-L63

## Summary
The `RapBattle::battle()` function relies on `block.timestamp` and `block.prevrandao` for randomness generation, a practice generally discouraged due to potential manipulation by calling contracts.

## Vulnerability Details
The predictability of `block.timestamp` and `block.prevrandao` allows attackers to calculate the outcome in advance. Specifically, on the Arbitrum network, `block.prevrandao` is always set to 1, as noted in the [Arbitrum documentation](https://docs.arbitrum.io/for-devs/concepts/differences-between-arbitrum-ethereum/solidity-support):
> block.prevrandao: Returns the constant 1. 

This constant value further exacerbates the issue of predictability.

## Impact
Attackers could exploit this vulnerability by determining the outcome of battles before participating, enabling them to choose battles they are certain to win or engage in front-running tactics to secure an advantage.

## Tools Used
Manual review.

## Recommendations
To mitigate this vulnerability, it is advisable to integrate a decentralized oracle for random number generation, such as [Chainlink`s VRF](https://docs.chain.link/vrf), which provides verifiable randomness that cannot be manipulated by participants or miners.
    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Reentrancy vulnerability in `OneShot::mintRapper()` allow users to temporarily gain additional skills, thereby increasing their chances of winning battles during that period            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/OneShot.sol#L29-L36

## Summary
The `OneShot::mintRapper()` function presents a reentrancy vulnerability due to its non-compliance with the Checks-Effects-Interactions (CEI) pattern. This flaw allows attackers to exploit the system by creating a contract specifically for minting new rappers. Attackers can then engage and win in battles against defenders with skill points under 65.

## Vulnerability Details
The issue stems from the `OneShot::mintRapper()` function executing `safeMint()` prior to updating the `RapperStats`, violating the CEI pattern. This order of operations allows for the minting of a rapper without proper initialization of their statistics:

```solidity
function mintRapper() public {
    uint256 tokenId = _nextTokenId++;
@>  _safeMint(msg.sender, tokenId);

     // Initialize metadata for the minted token
@>   rapperStats[tokenId] =
            RapperStats({weakKnees: true, heavyArms: true, spaghettiSweater: true, calmAndReady: false, battlesWon: 0});
}
```

However, due to improper initialization, the newly minted rapper's stats default to zero values, inadvertently assigning them 65 skill points:

```solidity
struct RapperStats {
        bool weakKnees;
        bool heavyArms;
        bool spaghettiSweater;
        bool calmAndReady;
        uint256 battlesWon;
    }
```

The resulting `RapperStats` of the tokenId we are minting will be `RapperStats({ weakKnees: false, heavyArms: false, spaghettiSweater: false, calmAndReady: false, battlesWon: 0 })`, giving us 65 points.

An attacker can deploy a contract that implements the `onERC721Received()` function, which is invoked by `_safeMint()` to ensure the recipient can accept ERC721 tokens. This contract can then initiate `RapBattle::goOnStageOrBattle()` against a defender with less than 65 skill points, leveraging the flawed stat assignment for an unfair advantage.

<details>
<summary>POC</summary>
The following addition to `OneShotTest.t.sol` demonstrates the exploit:

```solidity
contract MintAndAttack is IERC721Receiver {
    OneShot oneShot;
    RapBattle rapBattle;

    constructor(address oneShotAddress, address rapBattleAddress) {
        oneShot = OneShot(oneShotAddress);
        rapBattle = RapBattle(rapBattleAddress);
    }

    function attack() external {
        oneShot.mintRapper();
    }

    function onERC721Received(address, address, uint256 tokenId, bytes calldata) external override returns (bytes4) {
        oneShot.approve(address(rapBattle), tokenId);
        require(rapBattle.getRapperSkill(tokenId) > rapBattle.getRapperSkill(0), "I'm gonna lose");
        rapBattle.goOnStageOrBattle(tokenId, rapBattle.defenderBet());
        return IERC721Receiver.onERC721Received.selector;
    }
}
```

```solidity
function testMintAndAttackBattle() public {
        // 1. There is a defender on stage
        vm.startPrank(user);
        oneShot.mintRapper();
        oneShot.approve(address(rapBattle), 0);
        rapBattle.goOnStageOrBattle(0, 0);
        vm.stopPrank();

        // 2. Defender: 50 skill points
        //    Attacker: 65 skill points (default values on the struct before being updated after the mint)
        console.log("Defender Skill: ", rapBattle.getRapperSkill(0));
        console.log("Attacker Skill: ", rapBattle.getRapperSkill(1));

        // 3. Attacker see a defender on stage with less than 65 skill points, mint and go on stage to win the battle before the stats are updated
        vm.startPrank(challenger);
        MintAndAttack attackContract = new MintAndAttack(address(oneShot), address(rapBattle));
        attackContract.attack();
        vm.stopPrank();
    }
```
</details>


## Impact
This vulnerability allows an attacker to mint a rapper with artificially high skill points (equivalent to a 3-day stake) and exploit this to win battles against less skilled defenders.

## Tools Used
Manual review.

## Recommendations
To mitigate this vulnerability, the order of operations in `OneShot::mintRapper()` should be adjusted to adhere to the CEI pattern, ensuring proper initialization of rapper stats before the minting process:

```diff
function mintRapper() public {
        uint256 tokenId = _nextTokenId++;
-        _safeMint(msg.sender, tokenId);
+      // Initialize metadata for the minted token
+      rapperStats[tokenId] =
+           RapperStats({weakKnees: true, heavyArms: true, spaghettiSweater: true, calmAndReady: false, battlesWon: 0});

-       // Initialize metadata for the minted token
-        rapperStats[tokenId] =
-          RapperStats({weakKnees: true, heavyArms: true, spaghettiSweater: true, calmAndReady: false, battlesWon: 0});
+       _safeMint(msg.sender, tokenId);
    }
```

# Low Risk Findings

## <a id='L-01'></a>L-01. `RapBattle::_battle()` emits incorrect data on `Battle` event            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L67-L70

## Summary
The `RapBattle::_battle()` function potentially emits the `Battle` event with an incorrect winner due to a discrepancy in the winner determination logic.

## Vulnerability Details
The `RapBattle::battle()` function's documentation and logic aim to determine the battle winner based on the comparison between a randomly generated number (`random`) and the defender's skill level (`defenderRapperSkill`). The comment within the function specifies the intended logic:
```
// If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
```

This logic is correctly implemented in the conditional check:
```solidity
if (random <= defenderRapperSkill) {
```

However, the event that announces the battle's outcome incorrectly calculates the winner using a strict less-than comparison (`random < defenderRapperSkill`), as shown below:
```solidity
emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);
```

## Impact
Consider a scenario where both `random` and `defenderRapperSkill` equal 65. According to the intended logic, the defender should emerge victorious. However, due to the erroneous comparison in the event emission, the system mistakenly declares the challenger as the winner. This discrepancy can lead to external systems misinterpreting the outcome of battles, affecting game dynamics and participant strategies.

## Tools Used
Manual review.

## Recommendations
To resolve this inconsistency, adjust the event emission logic to align with the documented and implemented winner determination criteria:
```diff
- emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);
+ emit Battle(msg.sender, _tokenId, random <= defenderRapperSkill ? _defender : msg.sender);
```


