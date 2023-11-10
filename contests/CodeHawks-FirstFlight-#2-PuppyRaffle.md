# First Flight #2: Puppy Raffle - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. Bad RNG calculation cause the winner and the rarity to be predicted](#H-01)
  - ### [H-02. Missing player refunded check on the winner selection causing loss of funds](#H-02)
  - ### [H-03. Refunding more than one players cause no new players to enter after](#H-03)
  - ### [H-04. totalFees can overflow causing loss of funds for the owner](#H-04)
- ## Medium Risk Findings
  - ### [M-01. Sending a small amount of ETH causes the winner fees to not be collected](#M-01)
- ## Low Risk Findings
  - ### [L-01. Floating pragma can cause unexpected behavior](#L-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #2

### Dates: Oct 25th, 2023 - Nov 1st, 2023

[See more contest details here](https://www.codehawks.com/contests/clo383y5c000jjx087qrkbrj8)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 4
- Medium: 1
- Low: 1

# High Risk Findings

## <a id='H-01'></a>H-01. Bad RNG calculation cause the winner and the rarity to be predicted

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L139

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L129

## Summary

The RNG formula implemented in the `PuppyRaffle#selectWinner()` function for selecting the winner and the rarity is not following the best practices for generating random numbers on-chain causing the raffle to be corruptible.

## Vulnerability Details

The `PuppyRaffle#selectWinner()` function is using two bad methods to calculate a random number.
In the following line of code is trying to generate a random index to select the winner but the tree values used for the hash `msg.sender`, `block.timestamp` and `block.difficulty` are all visible for the public and can be easily obtained and tried being the `PuppyRaffle#selectWinner()` callable by anyone being `external`

```solidity
uint256 winnerIndex =
            uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
```

Same thing also for the rarity calculation

```solidity
uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100;
```

Anyone can call the function and see if the two parameters generate the rarity wanted.

## Impact

Any player can use the `PuppyRaffle#selectWinner()` function predicting the outcome of the raffle making some tries until the best outcome for them is generated causing the raffle to be unfair.

## Tools Used

Manual review.

## Recommendations

Use Chainlink VRF to have a better RNG method and make the raffle correct.

## <a id='H-02'></a>H-02. Missing player refunded check on the winner selection causing loss of funds

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L151

## Summary

The `PuppyRaffle#selectWinner()` function does not take into account the possibility that the winner could be a refunded player, therefore the address(0), causing 80% of the total in the pool to be burnt.

## Vulnerability Details

If a player get refunded the `PuppyRaffle#refund()` function will set the address in the `players` array at the specific index equal to the address(0). This will cause the following line of code in the `PuppyRaffle#selectWinner()` function to send the prize for the winner (if this player was selected as the winner) to be lost.

```solidity
(bool success,) = winner.call{value: prizePool}("");
```

## Impact

If the selected winner in a specific round is a refunded player funds will be lost being sent to the address(0)

## Tools Used

Manual review.

## Recommendations

Add a check in the `PuppyRaffle#selectWinner()` function making sure the winner will be an active player.

## <a id='H-03'></a>H-03. Refunding more than one players cause no new players to enter after

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L103

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L86-L90

## Summary

The function `PuppyRaffle#refund()` reset the address of the player refunded.
If two players get refunded, any subsequent call to the function `PuppyRaffle#enterRaffle()` will revert caused by the check on the duplicates and no new players will enter the raffle.

## Vulnerability Details

The following line in `PuppyRaffle#enterRaffle()` is checking for equality but it is not taking address 0 into account.

```solidity
require(players[i] != players[j], "PuppyRaffle: Duplicate player");
```

Having more than one address 0 in the `players` array is caused by the `PuppyRaffle#refund()` function at this particular line.

```solidity
players[playerIndex] = address(0);
```

A proof of concept of the attack is provided below.

```solidity
function testTwoRefundCauseNoNewPlayers() public playersEntered {
    vm.prank(playerOne);
    puppyRaffle.refund(0);

    vm.prank(playerTwo);
    puppyRaffle.refund(1);

    address playerFive = address(5);
    vm.deal(playerFive, 1 ether);
    address[] memory players = new address[](1);
    players[0] = playerFive;

    vm.expectRevert("PuppyRaffle: Duplicate player");
    puppyRaffle.enterRaffle{value: entranceFee * 1}(players);
}
```

## Impact

The previous situation can be caused either by a malicious user who requests a refund with 2 of his addresses or simply by two different players.

## Tools Used

Manual review.

## Recommendations

In the `PuppyRaffle#enterRaffle()` function make sure to take in consideration the address 0 when you are searching for duplicates.

## <a id='H-04'></a>H-04. totalFees can overflow causing loss of funds for the owner

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L30

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L134

## Summary

The storage variable `PuppyRaffle#totalFees` can overflow if fees are not withdrawn after a certain period and start accumulating.

## Vulnerability Details

Using a `uint64` for the storage variable `PuppyRaffle#totalFees` and not handling overflow manually due to the fact the solidity version you are using is `^0.7.6` can cause an overflow after `PuppyRaffle#totalFees` reach `type(uint64).max`, a value of 18.446.744.073.709.551.616 (~18 ether).

## Impact

The overflow of `PuppyRaffle#totalFees` cause a loss of funds for the owner because all the fees collected for every round will go to 0.

## Tools Used

Manual review.

## Recommendations

You have tree options to handle this:

- Use a bigger `uint` size, for example `uint256` to have enough space for the total fees collected
- Use the OpenZeppelin library SafeMath and handle the possible overflow in the `PuppyRaffle#selectWinner()` function
- Use a Solidity version >=0.8 that will throw an error for arithmetic overflow or underflow

# Medium Risk Findings

## <a id='M-01'></a>M-01. Sending a small amount of ETH causes the winner fees to not be collected

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L158

## Summary

The `PuppyRaffle#withdrawFees()` function is using `address(this).balance` to check if all the fees for the current round have been collected and therefore the round is over, but just a small amount of ETH added to the player fees cause the function to be DoS.

## Vulnerability Details

Using `address(this).balance` and trust that its value will always be synched with the specific smart contract logic is discouraged. In this case using `selfdestruct()` to send a small amount of ETH to the `PuppyRaffle` contract can cause the `address(this).balance` and the `totalFees` to always be out of sync causing a DoS on the `PuppyRaffle#withdrawFees()`.

## Impact

It is impossible for the owner to collect his fees because the `PuppyRaffle#withdrawFees()` will always revert.

## Tools Used

Manual review.

## Recommendations

Use a storage variable (bool or enum) to save the start and end of the raffle instead of relying on the fees collected and the balance.

# Low Risk Findings

## <a id='L-01'></a>L-01. Floating pragma can cause unexpected behavior

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L2

## Summary

The contract use a floating pragma statement and an old Solidity version.

## Impact

Using a floating pragma is discouraged as can cause different behavior based on the compiler version.
Also, an old version of Solidity can contain bugs and miss important optimizations applied in newer versions.

## Tools Used

Manual review.

## Recommendations

Use a stable pragma statement and the latest Solidity version.
