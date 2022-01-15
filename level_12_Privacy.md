# Level 12: Privacy

This is the level 12 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- [Fixed size byte arrays](https://docs.soliditylang.org/en/v0.8.7/types.html#fixed-size-byte-arrays)
- [Layout of State Variables](https://docs.soliditylang.org/en/v0.8.7/internals/layout_in_storage.html#) in a Solidity contract
- [Reading storage](https://web3js.readthedocs.io/en/v1.5.2/web3-eth.html#getstorageat) at a slot

## Hack
Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(now);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) public {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

`player` has to set `locked` state variable to `false`.

This is similar to level 8: Vault where any state variable (irrespective of whether it is `private`) can be read given it's slot number.

`unlock` uses the third entry (index 2) of `data` which is a `bytes32` array. Let's determined `data`'s third entry's slot number (each slot can accommodate at most 32 bytes) according to storage rules:

- `locked` is 1 byte `bool` in slot 0
- `ID` is a 32 byte `uint256`. It is 1 byte extra big to be inserted in slot 0. So it goes in & totally fills slot 1
- `flattening` - a 1 byte `uint8`, `denomination` - a 1 byte `uint8` and `awkwardness` - a 2 byte `uint16` totals 4 bytes. So, all three of these go into slot 2
- Array data always start a new slot, so `data` starts from slot 3. Since it is `bytes32` array each value takes 32 bytes. Hence value at index 0 is stored in slot 3, index 1 is stored in slot 4 and index 2 value goes into slot 5

Alright so key is in slot 5 (index 2 / third entry). Read it.

```javascript
key = await web3.eth.getStorageAt(contract.address, 5)

// Output: '0x5dd89f7b81030395311dd63330c747fe293140d92dbe7eee1df2a8c233ef8d6d'
```

This `key` is 32 byte. But `require` check in `unlock` converts the `data[2]` 32 byte value to a `byte16` before matching.

`byte16(data[2])` will truncate the last 16 bytes of `data[2]` and return only the first 16 bytes.

Accordingly convert `key` to a 16 byte hex (with prefix - `0x`):
```javascript
key = key.slice(0, 34)

// Output: 0x5dd89f7b81030395311dd63330c747fe
```

Call `unlock` with `key`:

```javascript
await contract.unlock(key)
```

Unlocked! Verify by:
```javascript
await contract.locked()

// Output: false
```

_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
