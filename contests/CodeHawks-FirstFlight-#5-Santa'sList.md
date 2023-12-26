# First Flight #5: Santa's List - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. Anyone can call `SantasList::buyPresent()` and burn the token of others to collect a present](#H-01)
  - ### [H-02. Incorrect Solmate library imported causing bad behaviors](#H-02)
  - ### [H-03. The first value of `SantasList::Status` is `NICE` making everyone `NICE` by default](#H-04)
  - ### [H-04. `SantasList::checkList()` is missing the `SantasList::onlySanta` modifier giving anyone the opportunity to change `Status`](#H-05)
- ## Medium Risk Findings
  - ### [M-01. `2e18` should be used as present cost and not `1e18`](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #5

### Dates: Nov 30th, 2023 - Dec 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clpba0ama0001ywpabex01hrp)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 4
- Medium: 1

# High Risk Findings

## <a id='H-01'></a>H-01. Anyone can call `SantasList::buyPresent()` and burn the token of others to collect a present

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L172C4-L175

## Summary

The `SantasList::buyPresent()` should be called only by the people with SantaTokens, so anyone `EXTRA_NICE` but in reality there is no check in the function.

## Vulnerability Details

The function `SantasList::buyPresent()` is used by the `EXTRA_NICE` people to buy a present for the `NAUGHTY` one burning the SantaTokens collected from the `SantasList::collectPresent()` and receiving a second NFT in exchange, but there is no control on the characteristics of the `msg.sender` and anyone can call it.

Add the following code in `SantasListTest.t.sol`:

```solidity
function testAnyoneCanBuyPresentFromOthers() public {
        address attacker = makeAddr("attacker");

        vm.startPrank(santa);
        santasList.checkList(user, SantasList.Status.EXTRA_NICE);
        santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
        vm.stopPrank();

        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

        vm.startPrank(user);
        santasList.collectPresent();
        assertEq(santasList.balanceOf(user), 1);
        assertEq(santaToken.balanceOf(user), 1e18);
        vm.stopPrank();

        vm.startPrank(attacker);
        santasList.buyPresent(user);
        assertEq(santasList.balanceOf(user), 1);
        assertEq(santaToken.balanceOf(user), 0);
        vm.stopPrank();
    }
```

## Impact

Anyone can see the SantaTokens balance of others and watch for token holders calling after that the `SantasList::buyPresent()`, burning their tokens without limitations and receiving in exchange an NFT as a present.

## Tools Used

Manual review.

## Recommendations

Make the following changes:

```diff
+ error SantasList__NotEnoughSantaTokens();

- function buyPresent(address presentReceiver) external {
+ function buyPresent() external {
+    if (i_santaToken.balanceOf(msg.sender) < PURCHASED_PRESENT_COST) {
+        revert SantasList__NotEnoughSantaTokens();
+   }
-   i_santaToken.burn(presentReceiver);
+  i_santaToken.burn(msg.sender);
    _mintAndIncrement();
}
```

## <a id='H-02'></a>H-02. Incorrect Solmate library imported causing bad behaviors

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantaToken.sol#L4

## Summary

The `SantaToken` contract import an incorrect Solmate library, maybe a corrupted one.

## Vulnerability Details

The following import statement

```solidity
import {ERC20} from "@solmate/src/tokens/ERC20.sol";
```

resolves to this one:

```solidity
import {ERC20} from "lib/solmate-bad/src/tokens/ERC20.sol";
```

This is not the correct Solmate library.

## Impact

The contract import a corrupted Solmate library causing bad and not expected behaviors.

## Tools Used

Manual review.

## Recommendations

Use the correct Solmate library that you can find at the following link [Solmate](https://github.com/transmissions11/solmate)

## <a id='H-03'></a>H-03. The first value of `SantasList::Status` is `NICE` making everyone `NICE` by default

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L69-L74

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L79-L80

## Summary

Each address in the two mappings, `SantasList::s_theListCheckedOnce` and `SantasList::s_theListCheckedTwice`, is automatically assigned the `NICE` status by default, as it represents the initial value in the `Status` enum.

## Vulnerability Details

In Solidity, a variable of a user-defined enum type does not have a null value. Instead, it defaults to the first value in the list of possible values, indexed at 0. In the `Status` enum, the first value is `NICE`. Consequently, any access to the two Santa's lists, `SantasList::s_theListCheckedOnce` and `SantasList::s_theListCheckedTwice`, will return a default value of `NICE` for individuals who have not yet been checked.

## Impact

The default assignment of a `NICE` status to everyone in the two lists, including those unknown or still not checked, makes them eligible for receiving the NFT present, unless specifically checked otherwise by Santa.

## Tools Used

Manual review.

## Recommendations

To address this, consider rearranging the enum `Status` as follows:

```javascript
enum Status {
    NOT_CHECKED_TWICE,
    EXTRA_NICE,
    NICE,
    NAUGHTY
}
```

By making `NOT_CHECKED_TWICE` the first value, you ensure a more logical default status for individuals who have not yet been checked.

## <a id='H-04'></a>H-04. `SantasList::checkList()` is missing the `SantasList::onlySanta` modifier giving anyone the opportunity to change `Status`

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L121-L124

## Summary

The `SantasList::checkList()` function is utilized by Santa to conduct a first pass on individuals, determining whether they fall under the "naughty" or "nice" category. Consequently, it is imperative that only Santa has the authority to invoke this function, and access should be restricted from everyone else.

## Vulnerability Details

The absence of the onlySanta modifier in the `SantasList::checkList()` function allows anyone to call the function without restriction. To rectify this vulnerability, the `SantasList::onlySanta` modifier should be incorporated into the `SantasList::checkList()` function, ensuring that only Santa can perform this crucial action.

## Impact

The current state of the `SantasList::checkList()` function permits unrestricted access, enabling anyone to invoke the external function and modify the status of themselves or others. This poses a significant security risk as it compromises the integrity of the list, allowing unauthorized individuals to influence the evaluation of naughty or nice status.

## Tools Used

Manual review.

## Recommendations

It is recommended to implement the following changes in the `SantasList::checkList() function signature:

```diff
- function checkList(address person, Status status) external {
+ function checkList(address person, Status status) external onlySanta {
        s_theListCheckedOnce[person] = status;
        emit CheckedOnce(person, status);
    }
```

By incorporating the `onlySanta` modifier, access to the `SantasList::checkList()` function will be restricted to Santa alone, mitigating the potential risks associated with unauthorized modifications to the list.

# Medium Risk Findings

## <a id='M-01'></a>M-01. `2e18` should be used as present cost and not `1e18`

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L88

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantaToken.sol#L25

https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantaToken.sol#L32

## Summary

`SantaToken::mint()` and `SantaToken::burn()` are using `1e18` respectively for the mint of the token when someone `EXTRA_NICE` collect the present and for burning it when someone buy the present, instead as described in the docs `2e18` should be the gift and the cost respectively.

## Impact

The implementation is not respecting the documentation and as a consequence the gift is smaller and also the price to buy a present for someone `NAUGHTY`.

## Tools Used

Manual review.

## Recommendations

```diff
function mint(address to) external {
        if (msg.sender != i_santasList) {
            revert SantaToken__NotSantasList();
        }
+        _mint(to, 2e18);
    }
```

```diff
function burn(address from) external {
        if (msg.sender != i_santasList) {
            revert SantaToken__NotSantasList();
        }
+        _burn(from, 2e18);
    }
```
