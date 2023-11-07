# First Flight #1: PasswordStore - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. setPassword missing owner check allowing everyone to call it ](#H-01)
    - ### [H-02. s_password can be read accessing EVM storage](#H-02)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #1

### Dates: Oct 18th, 2023 - Oct 25th, 2023

[See more contest details here](https://www.codehawks.com/contests/clnuo221v0001l50aomgo4nyn)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. setPassword missing owner check allowing everyone to call it             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/7a2fc760235c4f4809511186ff9a061c2ec68634/src/PasswordStore.sol#L26-L29

## Summary

The `setPassword` function should be callable by only the owner but check on the caller is missing.

## Vulnerability Details

The function `setPassword` is external and is missing any check on the `msg.sender`, so anyone can call it.
Here is a Proof of Concept unit test demonstrating the issue (add it to `PasswordStoreTest.t.sol`):
```solidity
function test_non_owner_can_set_password() public {
        vm.prank(owner);
        string memory ownerPassword = "ownerPassword";
        passwordStore.setPassword(ownerPassword);

        vm.prank(address(1));
        string memory userPassword = "userPassword";
        passwordStore.setPassword(userPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();

        assertEq(actualPassword, userPassword);
    }
```

## Impact

Anyone can change the owner stored password overriding the previous one, updating it or deleting it.

## Tools Used

Manual review.

## Recommendations

You can implement in `setPassword()` the same check you have in `getPassword()`.

```diff
function setPassword(string memory newPassword) external {
+       if (msg.sender != s_owner) {
+           revert PasswordStore__NotOwner();
+       }
        s_password = newPassword;
        emit SetNetPassword();
    }
```

Or better, now that you have the same code in two functions create a modifier with this check.
```solidity
modifier onlyOwner() {
        if (msg.sender != s_owner) {
            revert PasswordStore__NotOwner();
        }
        _;
    }
```

Now the two function signatures become:
```solidity
function setPassword(string memory newPassword) external onlyOwner

function getPassword() external view onlyOwner returns (string memory)
```
## <a id='H-02'></a>H-02. s_password can be read accessing EVM storage            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/7a2fc760235c4f4809511186ff9a061c2ec68634/src/PasswordStore.sol#L14

## Summary

The state variable `s_password` is private but the value can still be accessed reading the specific storage slot (slot 1 in this case).

## Vulnerability Details

The password is readable on-chain, hence, can be accessed anytime.
Making it private don't prevent it from being read externally.
Here is a Proof of Concept unit test demonstrating the issue (add it to `PasswordStoreTest.t.sol`):

```solidity
    function test_non_owner_can_read_password() public {
        vm.prank(owner);
        string memory expectedPassword = "myNewPassword";
        passwordStore.setPassword(expectedPassword);
        vm.prank(address(1));
        // We access the slot 1 of the storage where the password resides and we get the first 13 bytes (expectedPassword length)
        bytes13 slotOneValue = bytes13(vm.load(address(passwordStore), bytes32(uint256(1))));
        string memory actualPassword = string(abi.encodePacked(slotOneValue));
        assertEq(actualPassword, expectedPassword);
    }
```

## Impact

Reading the password gives the attacker the ability to access funds or sensitive information.

## Tools Used

Manual review.

## Recommendations

Storing a password on-chain is considered bad practice.
Everything you put on-chain is visible.
You should not store it in plaintext or eventually store the encrypted or hashed version.
But other problems arise such as rainbow tables ecc.
		





