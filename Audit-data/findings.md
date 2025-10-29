### [S-#] Storing the password on-chain makes it visible to anyone, not a private password

**Description:** All data stored on-chain is visble to anyone, and can be directly read from the blockchain. 
The `PasswordStore::s_password` is intended to be private variable which can be accessed only through `PasswordStore::getPassword` function which is intended to be called by only the owner of the contract.

**Impact:** Anyone can read the password and severely break the functionality of the protocol

**Proof of Concept:** (Proof of Code)

The below test case shows how anyone could read the password directly from the blockchain. We use [foundry's cast](https://github.com/foundry-rs/foundry) tool to read directly from the storage of the contract, without being the owner. 

1. Create a locally running chain
```bash
make anvil
```

2. Deploy the contract to the chain

```
make deploy 
```

3. Run the storage tool

We use `1` because that's the storage slot of `s_password` in the contract.

```
cast storage <CONTRACT_ADDRESS> 1 --rpc-url http://127.0.0.1:8545
```

You'll get an output that looks like this:

`0x6d7950617373776f726400000000000000000000000000000000000000000014`

You can then parse that hex to a string with:

```
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

And get an output of:

```
myPassword
```

**Recommended Mitigation:** 
Due to this, the overall architecture of the contract should be rethought. One could encrypt the password off-chain, and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. However, you'd also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with the password that decrypts your password. 


### [S-#] `PasswordStore::setPassord` function has no access controls. Meaning a non-owner can set the new password.

**Description:** The `PasswordStore::setPassword` function is set to be an external function, however natspec and the overall purpose of the smart contract is `This function allows only the owner to set a new password`.

```javascript
function setPassword(string memory newPassword) external {
@>      //@ audit-high There is no access controls
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:** Anyone can set/change the password, severely breaking the contract functionality.

**Proof of Concept:** Add the following to `PasswordStore.t.sol` test file

<details>
<summary>Code</summary>

```javascript
    function test_anyone_can_set_password(address randomUser) public {
        vm.assume(randomUser != owner);
        vm.prank(randomUser);
        string memory expectedPassword = "userpassword";
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(actualPassword, expectedPassword);
    }
```

</details>


**Recommended Mitigation:** Add an access control conditional to the `setPassword` function.

```javascript
if(msg.sender != s_owner){
    revert PasswordStore__NotOwner();
}
```

### [S-#] The natspec indicates there is `@param newPassord` a parameter that does not exist, meaning natspec is incorrect

**Description:** 

```javascript
/*
     * @notice This allows only the owner to retrieve the password.
     * @param newPassword The new password to set.
     */
    
    function getPassword() external view returns (string memory) {
        if (msg.sender != s_owner) {
            revert PasswordStore__NotOwner();
        }
        return s_password;
    }
```
The `PasswordStore::getPassword` signture is `getPassword()` but the natspec indicates that it should be `getPassword(string)`.

**Impact:** The natspec is incorrect


**Recommended Mitigation:** Remove the incorrect natspec line

```diff
- @param newPassword The new password to set.
```
